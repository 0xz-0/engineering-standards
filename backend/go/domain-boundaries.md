# Go 领域依赖治理规范

> **约束力**：MUST
> **适用范围**：所有 Go 编写的服务（微服务与 BFF）中，`internal/` 下包含多个领域（domain）的场景；顶层目录、六层骨架与服务分类见 [`project-layout.md`](project-layout.md)。
> **说明**：本文规定一个领域内两类接口的定义位置、多领域协作与隔离规则，以及如何用自动化测试强制校验依赖边界。

## 1. 接口（端口）定义位置

一个 `domain` 内部会出现两类接口，定义位置不同，**禁止**混淆：

1. **持久化 / 技术依赖端口**：定义在 `internal/domain/<domain>` 下，核心持久化端口**必须**命名为 `Gateway`；方法签名直接使用该领域的 domain 类型，描述"如何存取本领域的核心事实"；由 `internal/infrastructure/<能力>/<domain>` 实现，`application` 层持有并调用，**不**关心具体实现（PostgreSQL、Redis 等）。同一领域如需为其他技术依赖（如独立的缓存访问）拆出额外端口，以能力前缀命名（如 `CacheGateway`），核心持久化端口仍固定命名为 `Gateway`。

   ```go
   // 领域端口：由 infrastructure 层实现
   type Gateway interface {
       Get(context.Context, string, uint64) (Cart, bool, error)
       Create(context.Context, *Cart) error
       Update(context.Context, Cart, int) error
   }
   ```

2. **应用契约端口**：定义在 `internal/application/<domain>` 下，又分两种：
   - **对外暴露的应用契约**（供 `adapter` 或其他领域调用）：**必须**以 `XxxApplication` 命名；方法的输入类型按操作性质命名——写操作以 `Command` 结尾、读操作以 `Query` 结尾，返回的聚合读模型以 `View` 结尾；**不**向调用方泄漏 domain 内部类型；
   - **对其他领域能力的依赖抽象**：命名**必须**清晰表达该外部能力本身（如 `CatalogReader`、`OrderPlacer`），**禁止**使用 `Helper`、`Manager`、`Util` 等笼统命名；每个接口**只**覆盖单一职责的外部能力，**禁止**将多个不相关能力合并到同一接口；输入输出类型的命名规则与上一条相同，**不**直接引用对方领域的 domain 或 application 类型。

   ```go
   // 对外应用契约
   type CartApplication interface {
       Get(context.Context, CartQuery) (CartView, error)
       SetItem(context.Context, SetItemCommand) (CartView, error)
   }

   // 对外部能力的依赖抽象
   type CatalogReader interface {
       GetForCart(context.Context, CatalogQuery) (CatalogItem, error)
   }
   ```

3. **依赖抽象的实现位置**：领域内自身的持久化端口（第 1 类）由 `internal/infrastructure` 实现；**跨领域**的依赖抽象（第 2 类第二种）由 `internal/bootstrap` 下的防腐层（Anti-Corruption Layer，ACL）适配器实现，在其中完成两个领域 DTO 之间的转换，领域本身**不**感知协作对象的具体类型（详见第 2 章）。若领域间协作使用领域事件（第 2 章第 4 条），事件发布端口同样定义在 `domain` 层、命名与位置规则与第 1 点一致，由 `infrastructure` 层实现；事件投递的可靠性保证（至少一次投递、事务性 Outbox 等）按业务场景设计，不在本文强制范围内。

## 2. 多领域协作与隔离规则

当一个服务内部包含多个业务边界清晰的领域时（无论是微服务承载的多条业务能力，还是 BFF 自身沉淀的多个产品用例域，如 `focus` / `sync` / `backup` / `feedback`），均需遵守以下规则，以便未来需要将某个领域拆分为独立微服务时，只需替换跨进程通信适配器，不必改动业务代码：

1. 领域内部实现（`domain`、`application`、`adapter` 的非导出类型及未在第 1 章第 2 类中定义为对外契约的类型）**禁止**被其他领域直接 import；
2. 领域间同步协作**只能**经由目标领域在 `application` 层暴露的对外契约（第 1 章第 2 类第一种）调用，**禁止**绕过该契约直接访问目标领域内部；
3. 领域间的类型转换统一在 `internal/bootstrap` 下的 ACL 适配器中完成（第 1 章第 3 点），领域本身不做跨领域类型转换；
4. 领域间协作优先使用领域事件（异步）或第 2 条的对外契约（同步调用）；**禁止**跨领域直接调用私有方法。对强一致性要求高、调用量不大的场景，**可以**多个领域共享同一本地事务（如通过 context 传递同一 DB session），但必须同时满足：
   - 事务范围内**禁止**包含跨进程调用（下游微服务、第三方渠道、消息队列等网络请求），避免持有数据库锁等待网络 I/O 造成长事务；
   - 事务涉及的多领域数据行**必须**按稳定顺序读取 / 加锁，避免死锁；
   - 调用量增长、或该领域后续需要独立部署时，**必须**将共享事务重构为幂等 Saga 配合事务消息（Outbox）或其他分布式事务方案，不再依赖跨领域共享事务；
5. 数据库表按领域归属，即使多个领域共用同一数据库实例，**一张表只能归属一个领域**；
6. 共享的缓存 / 消息队列 key **必须**以 `:` 分隔，前两段固定为服务名与领域名（`<服务名>:<领域名>:...`），避免不同领域间键冲突；
7. `domain/shared`（或等价的共享目录）**只**存放跨领域稳定共享的概念（如统一的货币值对象、业务时钟），**禁止**沉淀为通用工具包；
8. **只有** `internal/bootstrap` 允许同时依赖多个领域的内部实现，用于完成组合与 ACL 转换。

第 3 章描述如何用自动化测试强制校验第 1、8 两条规则。

## 3. 依赖边界自动化校验

**MUST**：只要服务的 `internal/` 下存在多个领域（即触发第 2 章），必须在 `internal/architecture` 建立基于 `go test` 的架构测试，随 `go test ./...` 一起在 CI 中运行；**禁止**为此另外维护一个独立的命令行校验工具——用普通单测覆盖，才能保证每次改动都被本地和 CI 的常规测试流程强制检查到，不依赖额外的执行步骤。

### 3.1 机制

1. 用 `go list -json ./...` 获取每个包的 `ImportPath`、`Imports`（直接依赖）与 `Deps`（完整依赖闭包）；
2. 按包路径中的目录名判断该包归属哪个领域；
3. **同时**检查 `Imports` 与 `Deps` 两个集合，**不能**只检查直接 import——否则可以借助一个"看似中立"的内部包间接引入被禁止的跨领域依赖，从而绕过校验；
4. 除 `internal/bootstrap` 外的包，只要依赖闭包中出现了不属于自己领域的其他领域包，测试即失败，并报告是哪个包依赖了哪个领域；
5. 对 `internal/bootstrap` 的识别**必须**按包路径精确匹配（`== 该路径` 或 `HasPrefix(该路径+"/")`），**禁止**用名称前缀模糊匹配，防止 `internal/bootstrapper` 这类"形似但不是"的包被误判为豁免；
6. 同一 `internal/architecture` 包内，还应覆盖 [`project-layout.md`](project-layout.md) 第 4 章规则 1（`domain` 不得依赖 `infrastructure` 或框架 SDK）的校验：对 `domain`、`application` 包的 `Imports` 做禁止关键字/包路径匹配（如 `gorm.io/`、具体 HTTP 框架、第三方 SDK 包名关键字），命中即失败。

### 3.2 代码骨架

```go
package architecture

// domains 登记服务内的全部领域目录名，新增领域时需要同步登记。
var domains = map[string]struct{}{
    "cart": {}, "order": {}, "inventory": {}, // ...
}

type goPackage struct {
    ImportPath string
    Imports    []string // 直接依赖
    Deps       []string // 完整依赖闭包
}

// domainOf 从包路径中提取其所属领域目录名，非领域内的公共包返回空字符串。
func domainOf(importPath string) string { /* 按模块前缀 + 路径分段匹配 domains */ }

// isBootstrapPackage 只用路径精确匹配，不用前缀猜测。
func isBootstrapPackage(importPath string) bool {
    const bootstrapPath = "<module>/internal/bootstrap"
    return importPath == bootstrapPath || strings.HasPrefix(importPath, bootstrapPath+"/")
}

// crossDomainDependencies 汇总一个包的 Imports+Deps 中，跨越到其他领域的依赖。
func crossDomainDependencies(pkg goPackage) map[string]string {
    if isBootstrapPackage(pkg.ImportPath) {
        return nil // 唯一豁免
    }
    owner := domainOf(pkg.ImportPath)
    result := make(map[string]string)
    for _, imported := range append(append([]string(nil), pkg.Imports...), pkg.Deps...) {
        if target := domainOf(imported); target != "" && target != owner {
            result[imported] = target
        }
    }
    return result
}

func TestDomainsDoNotImportOtherDomainInternals(t *testing.T) {
    // go list -json ./... -> 逐包调用 crossDomainDependencies，任何非空结果都 t.Errorf。
}

func TestDomainAndApplicationDoNotImportInfrastructureOrForbiddenSDK(t *testing.T) {
    // go list -json ../domain/... ../application/... -> 对 Imports 做禁止关键字匹配。
}
```

## 4. 参考资料

- [`project-layout.md`](project-layout.md)：顶层目录结构、内部六层骨架、服务分类与差异对照；
- [`backend/go/coding-style.md`](coding-style.md)：命名与代码组织约定。
