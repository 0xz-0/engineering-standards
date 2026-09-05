# Go 项目结构规范

> **约束力**：MUST
> **适用范围**：所有 Go 编写的服务，含微服务与 BFF 两类；Node.js / TypeScript 编写的 BFF 服务见 [`bff/project-layout.md`](../../bff/project-layout.md)。
> **说明**：本文统一 Go 服务的顶层目录、内部分层与服务分类；跨领域的接口定义位置、隔离规则与自动化依赖校验见 [`domain-boundaries.md`](domain-boundaries.md)。

## 1. 背景与目的

团队现有 Go 服务已经自发收敛到同一套分层骨架（`domain` / `application` / `adapter` / `infrastructure` / `interface` / `bootstrap`），但在应用层目录命名、依赖边界校验方式、组合根位置等细节上，微服务与 BFF 两类服务出现了分叉。本规范把已验证可行的做法固定下来，并统一分叉点，作为新建服务的强制起点、存量服务重构时的对齐目标。

## 2. 服务分类

Go 服务分为两类，目录结构共享同一套分层骨架（见第 4 章），差异体现在职责边界和第 5 章描述的细节上。

### 2.1 微服务（Microservice）

- **定位**：面向多条业务线共用的能力中心，如商品交易（catalog / cart / order / inventory / payment 等）、用户身份与权益。同一个服务内部可能划分多个领域（domain），分别对应一套 `domain` / `application` / `adapter` 目录（见第 4 章），以单仓库、单进程部署（modular monolith），供多个 BFF 或其他微服务调用。
- **判断标准**：服务的调用方是多条产品线，而非单一客户端；服务对外只暴露稳定的业务能力接口，不感知具体产品线的客户端形态。
- **职责**：拥有并持久化核心业务事实（如订单、支付、用户身份），是这些事实的唯一权威来源。

### 2.2 BFF（Backend For Frontend）

- **定位**：面向单一产品线的转发与编排层，服务对象是这一条产品线下的全部客户端（如同一 App 的 iOS、Android、WinPC、macOS、Web 等多端），而非某一条产品线的单一客户端形态。
- **判断标准**：服务的调用方限定在一条产品线内（可以是该产品线的多个客户端），不对外部产品线提供能力；不持久化属于其他微服务的核心业务事实，而是通过内网同步调用转发。
- **职责**：
  - 承接该产品线**自有**的、不与其他产品线共享的数据与用例（如本地业务功能、离线同步、备份等）；
  - 将登录、账号、交易、权益等跨产品线共用的能力转发给对应微服务，在边界上做协议适配（错误码映射、鉴权头透传等），**不重新实现或缓存这些核心业务事实**。

### 2.3 协作方式

- BFF 通过 `internal/infrastructure/gateway/<微服务名>` 下的客户端调用微服务，调用方式为内网同步 HTTP；如需使用其他同步协议（如 gRPC），须在服务的 `docs/architecture.md` 中说明选型理由；
- 微服务之间的协作规则见 [`domain-boundaries.md`](domain-boundaries.md) 第 2 章「多领域协作与隔离规则」；
- **禁止**微服务反向依赖某个 BFF；**禁止**BFF 之间直接互相调用（如确有需要，应上升为微服务能力）。

## 3. 顶层目录结构

| 目录 | 职责 | 必要性 | 说明 |
| --- | --- | --- | --- |
| `cmd/` | 可执行入口 | MUST | 默认单进程入口（仅 `cmd/server`）；仅当存在需要独立部署运行的异步处理（worker）、一次性运维工具等场景时，才允许拆分为多个二进制，每个子目录对应一个独立进程。`main.go` 只做进程启动（读配置、构建 `bootstrap`、`Run`、优雅关闭），**禁止**在此编写业务逻辑或依赖装配细节 |
| `internal/` | 全部业务代码 | MUST | Go 语言级强制不可被其他仓库导入；内部分层见第 4 章 |
| `configs/` | 配置模板/示例 | MUST | 提交仓库的只是模板与本地开发默认值，真实配置由配置中心或环境变量注入，详见 [configuration.md](configuration.md) |
| `deploy/` | 部署清单 | MUST | Dockerfile、K8s manifest 等部署产物 |
| `docs/` | 服务文档 | MUST | 须包含 `architecture.md`，描述本服务的分层、依赖方向与（如适用）领域划分；可按需拆分 `docs/architecture/*.md` 存放专题边界说明 |
| `tools/` | 内部开发工具 | MUST | 代码生成器、数据库迁移辅助脚本等不随主二进制发布的工具；依赖边界校验**不**属于此类，须按 [`domain-boundaries.md`](domain-boundaries.md) 放在 `internal/architecture` 下 |
| `migrations/` + `cmd/migrate` | 数据库迁移 | SHOULD | 当服务独立拥有数据库 schema 的演进权时，建议在顶层新增 `migrations/` 并配一个 `cmd/migrate` 入口管理迁移；是否需要取决于服务实际的 schema 归属（见第 5 章差异对照），不强制每个服务都有 |
| `openspec/`（或团队采用的规格驱动开发工具目录） | 规格驱动开发产物 | MAY | 如采用规格驱动开发（spec-driven development）工具，其提案/规格产物建议放在顶层独立目录，不影响运行时代码结构 |

## 4. 内部分层与依赖方向

`internal/` 下按以下六个包划分职责：

| 层 | 目录 | 职责 |
| --- | --- | --- |
| 领域层 | `internal/domain/<domain>` | 领域模型、值对象、领域事件，以及**领域端口**（命名与定义规则见 [`domain-boundaries.md`](domain-boundaries.md) 第 1 章）。不依赖 Gin、GORM、Redis 等任何技术框架或 SDK，不依赖 `infrastructure` |
| 应用层 | `internal/application/<domain>` | 用例编排：组织领域对象完成一次业务操作，通过领域端口调用基础设施，不感知具体技术实现，不感知 HTTP 等协议细节 |
| 适配层 | `internal/adapter/<domain>` | 面向具体协议的入站适配器（handler / controller 及其 DTO），负责协议数据与应用层命令/查询之间的转换 |
| 接口层 | `internal/interface/http` | 全局 HTTP 装配：路由注册、通用中间件（鉴权、日志、trace、错误响应包装）、启动与优雅关闭；**不**包含具体业务领域逻辑 |
| 基础设施层 | `internal/infrastructure/<能力>` | 技术实现：数据库、缓存、消息队列、第三方/下游服务网关（`gateway/<下游服务名>`）、配置加载、日志、指标等，实现领域端口 |
| 组合根 | `internal/bootstrap` | 唯一组合根，负责按依赖顺序构造并注入各层实例；是**唯一**允许同时引用多个层、多个 `domain` 的包 |

依赖方向：

```text
interface/http, adapter  →  application  →  domain
infrastructure           →  domain（实现其定义的端口）
bootstrap                →  domain / application / adapter / infrastructure（全部）
```

**MUST 规则**：

1. `domain` 包**禁止** import 任何 `infrastructure` 包或第三方框架 SDK；
2. `application` 包**禁止**直接 import `infrastructure` 的具体实现包，只能依赖 `domain` 定义的端口（接口）；
3. `adapter`、`interface/http` **禁止**跳过 `application` 直接调用 `infrastructure`；
4. 除 `internal/bootstrap` 外，其余任何包**禁止**同时依赖两个不同 `domain` 的内部实现（多领域场景下的详细规则见 [`domain-boundaries.md`](domain-boundaries.md)）。

## 5. 微服务与 BFF 的实践差异

第 3、4 章的目录结构与分层规则对两类服务完全一致。以下是两类服务在实践中会有出入、需要按各自定位取舍的地方：

| 维度 | 微服务 | BFF |
| --- | --- | --- |
| 对外契约的稳定性要求 | [`domain-boundaries.md`](domain-boundaries.md) 定义的对外应用契约会被多条产品线消费，变更**必须**考虑向后兼容与版本化，参见 [`general/versioning.md`](../../general/versioning.md) | 对外契约的调用方限定在自己一条产品线的多端客户端内，兼容性影响范围小于微服务，但仍需遵守同样的分层与契约规则 |
| 核心业务事实归属 | 拥有并持久化核心业务事实，是唯一权威来源 | **禁止**为已由某个微服务承担的核心业务事实（登录态、交易、权益等）重复建模持久化，只允许持久化"自身产品线独有"的数据（见第 2.2 节） |
| 调用其他服务的位置 | 通过 `internal/bootstrap` 下的 ACL 适配器调用同进程内其他领域；确需跨进程调用其他微服务时，走 `internal/infrastructure/gateway/<对方服务名>` | 通过 `internal/infrastructure/gateway/<微服务名>`（如 `gateway/center`）同步调用所依赖的微服务，在 `adapter` 层完成协议映射 |
| 数据库迁移落地方式 | `tools/db_migrations` 下的 up/down SQL + 测试校验，交由团队统一的数据库变更流程执行，不内置 `cmd/migrate` | 独立的 `migrations/` 目录 + `cmd/migrate` 二进制自行执行迁移 |

**数据库迁移方式选择**：两种方式均被本规范允许，是否需要独立的 `migrations/` + `cmd/migrate`（而非交由团队统一的数据库变更流程）取决于服务是否独立拥有数据库 schema 的演进权，不强制统一为一种；核心要求是迁移脚本必须成对提供 up/down 且有测试覆盖迁移链完整性。

## 6. 完整目录树示例

以下两份目录树均省略了非本文关注的通用文件（`.gitignore`、`README.md` 等），只保留与分层、依赖相关的结构，供新建服务参照。

### 6.1 微服务示例（以两个领域 `catalog`、`order` 为例）

```text
service-name/
├── cmd/
│   └── server/
│       └── main.go              # 只做进程启动：读配置、构建 bootstrap、Run、优雅关闭
├── configs/
│   └── config.example.yaml
├── deploy/
├── docs/
│   └── architecture.md          # 描述分层、依赖方向与领域划分
├── internal/
│   ├── domain/
│   │   ├── catalog/
│   │   │   ├── product.go
│   │   │   ├── gateway.go       # 持久化端口，由 infrastructure/gateway/catalog 实现
│   │   │   └── doc.go
│   │   └── order/
│   │       ├── order.go
│   │       ├── gateway.go
│   │       └── doc.go
│   ├── application/
│   │   ├── catalog/
│   │   │   ├── ports.go         # 对外应用契约
│   │   │   ├── service.go
│   │   │   └── doc.go
│   │   └── order/
│   │       ├── ports.go         # 对外应用契约 + 对 catalog 能力的依赖抽象（CatalogReader 等）
│   │       ├── service.go
│   │       └── doc.go
│   ├── adapter/
│   │   ├── catalog/
│   │   │   ├── handler.go
│   │   │   └── dto/
│   │   └── order/
│   │       ├── handler.go
│   │       └── dto/
│   ├── interface/
│   │   └── http/
│   │       ├── server.go
│   │       ├── middlewares.go
│   │       └── controller.go
│   ├── infrastructure/
│   │   ├── config/
│   │   ├── database/
│   │   └── gateway/
│   │       ├── catalog/         # 实现 domain/catalog.Gateway
│   │       └── order/           # 实现 domain/order.Gateway
│   ├── bootstrap/
│   │   ├── composition.go       # 唯一组合根
│   │   ├── order_catalog_adapter.go  # ACL：order 对 catalog 能力依赖抽象的实现
│   │   └── http_routes.go
│   └── architecture/
│       └── dependencies_test.go # 依赖边界自动化校验，见 domain-boundaries.md
├── tools/
│   └── db_migrations/
├── go.mod
└── go.sum
```

### 6.2 BFF 示例（自有领域 `focus`、`sync`，转发登录 / 交易能力给微服务）

```text
product-bff/
├── cmd/
│   ├── server/
│   │   └── main.go
│   ├── worker/                  # 异步处理确需独立部署时才拆出（见第 3 章）
│   │   └── main.go
│   └── migrate/
│       └── main.go
├── configs/
├── deploy/
├── docs/
│   └── architecture.md
├── migrations/
│   ├── 0001_focus.up.sql
│   └── 0001_focus.down.sql
├── internal/
│   ├── domain/
│   │   ├── focus/
│   │   └── sync/
│   ├── application/
│   │   ├── focus/
│   │   └── sync/
│   ├── adapter/
│   │   ├── focus/
│   │   └── sync/
│   ├── interface/
│   │   └── http/
│   ├── infrastructure/
│   │   ├── config/
│   │   ├── postgresql/
│   │   ├── queue/
│   │   └── gateway/
│   │       ├── identity_svc/     # 转发登录、账号、权益能力
│   │       └── trading_svc/      # 转发商品、订阅、支付能力
│   ├── bootstrap/
│   └── architecture/
│       └── dependencies_test.go
├── tools/
├── go.mod
└── go.sum
```

## 7. 参考资料

- [`domain-boundaries.md`](domain-boundaries.md)：接口（端口）定义位置、多领域协作与隔离规则、依赖边界自动化校验；
- [`backend/go/coding-style.md`](coding-style.md)、[`backend/go/error-handling.md`](error-handling.md)、[`backend/go/logging.md`](logging.md)、[`backend/go/configuration.md`](configuration.md)：与本文分层配套的编码、错误处理、日志、配置规范；
- [`general/versioning.md`](../../general/versioning.md)：微服务对外契约的版本化规则；
- [`bff/project-layout.md`](../../bff/project-layout.md)：Node.js / TypeScript 编写的 BFF 服务结构规范。
