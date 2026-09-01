# 数据库命名规范

> **约束力**：MUST
> **适用范围**：所有业务关系型数据库（PostgreSQL / MySQL 等）
> **说明**：本文约定跨数据库通用的命名原则；各数据库的落地差异（保留字清单、长度上限等）见对应子目录规范。

## 1. 基础风格

| 规则项 | 规范内容 |
| :--- | :--- |
| **命名风格** | 全小写 + `snake_case`（单词间用单下划线 `_`） |
| **表名数制** | **强制单数**（如 `product`，非 `products`）。单复数看**单条记录**的语义，不看全表数据量 |
| **分隔符（索引专用）** | **双下划线 `__`**，用于区隔"前缀"、"表名"、"字段名"，如 `idx__order_info__user_id` |
| **缩写纪律** | **严禁**自行发明缩写（各人习惯不同，命名将失去可预测性）。只允许使用各数据库专项规范中缩写对照表收录的词；确需新增时，先提 PR 补充对照表再使用 |

## 2. 字段语义分层（核心）

**原则：每个字段的名称必须反映其"业务语义"，而非"物理存储类型"。** 禁止将系统主键直接暴露给用户侧。

| 字段命名 | 语义定义 | 适用场景 | 展示范围 |
| :--- | :--- | :--- | :--- |
| **`id`** | 系统内部唯一标识符，数据库自增的技术主键 | 1. 表的主键<br>2. 逻辑外键关联（如 `user_id`）<br>3. 后台管理内部 JOIN | **严禁**直接展示给用户（如 URL、API 响应体） |
| **`no`** 或 **`code`** | 业务编号，具有明确业务含义，用于用户识别、客服查询、外部系统对接 | 用户可见的订单号、流水号、发票号码、物流运单号 | **必须**展示给用户，作为业务沟通的唯一凭证 |

### 🚫 红线规则（强制）

- **禁止将 `id` 暴露给用户**：URL 路径 `GET /order/12345` 中的 `12345` 严禁是自增 `id`（信息泄露 + 业务耦合）。正确做法：`GET /order/ORD20260818001`。
- **`id` 对外输出必须经应用层统一编码**：后台管理等确需按内部 `id` 定位的场景，对外输出时必须经应用层**不可逆编码**（如 Hashids / Base62）或直接映射为业务编号 `no`。**编码方案全公司统一，严禁各项目自造**（否则跨系统对接时同一 id 多种编码，失去规范意义）。
- **禁止将 `no` 用作物理外键关联**：外键必须指向不可变的主键 `id`（如 `order_info.user_id` 必须关联 `user_info.id`，不能关联 `user_info.user_no`）。

## 3. 字段后缀语义约定

字段后缀表达业务语义，全公司统一：

| 后缀 | 语义定义 | 数据类型 | 示例 |
| :--- | :--- | :--- | :--- |
| `_name` | 名称，用户可读，通常可更新 | 字符串 | `product_name`, `user_name` |
| `_code` | 编码，通常用于枚举或分类，不可变 | 字符串 | `country_code`, `category_code` |
| `_type` | 类型分类，区分同一实体下的子类 | 字符串（TEXT 枚举） | `payment_type`, `user_type` |
| `_status` | 状态，生命周期阶段，通常有状态机驱动 | 字符串（TEXT 枚举） | `order_status`, `shipment_status` |
| `_cents` | 金额（单位：分，仅限固定两位小数的单一/已知币种场景） | 大整数 | `total_cents`, `unit_price_cents` |
| `_minor` | 金额（该货币最小货币单位，多币种场景专用，需配合 `currency_code` + `currency_exponent` 换算，见[表设计规范 2.2](table-design.md#22-多币种例外commerce-multi-currency-exceptionmay)） | 大整数 | `amount_minor` |
| `_exponent` | 小数位指数（如 ISO 4217 币种指数） | 整数 | `currency_exponent` |
| `_at` | 时间点 | 时间戳 | `paid_at`, `shipped_at`, `canceled_at` |

**字段前缀 `provider_`**：表示该字段为第三方/上游渠道返回的原始值快照，仅用于对账溯源，**禁止**作为业务计算或展示依据（示例：`provider_amount_milliunits`，见[表设计规范 2.2](table-design.md#22-多币种例外commerce-multi-currency-exceptionmay)）。

## 4. 逻辑外键命名

- **格式**：`[被引用表名单数]_id`
- **示例**：`order_info` 引用 `user_info` → `user_id`；`order_product` 引用 `order_info` → `order_id`

## 5. 关联表（多对多中间表）命名

| 场景 | 命名格式 | 示例 |
| :--- | :--- | :--- |
| **纯关联表**（仅含两个外键 + 审计字段，无额外业务字段） | **推荐**使用 `[表A]_[表B]_relation` | `user_role_relation`（仅含 `user_id`、`role_id` 两个外键，无额外业务字段） |
| **带业务属性的关联表**（含数量、价格、状态、权重等） | **禁止**使用 `_relation`，保持简洁的 `[表A]_[表B]` | `order_product`（存 `quantity`、`unit_price_cents`）<br>`user_coupon`（存 `usage_status`、`used_at`） |

## 6. 索引命名

| 索引类型 | 前缀 | 格式 | 示例 |
| :--- | :--- | :--- | :--- |
| **普通索引** | `idx` | `idx__[表名]__[字段1]__[字段2]` | `idx__order_info__user_id__created_at` |
| **唯一索引** | `uniq` | `uniq__[表名]__[字段1]__[字段2]` | `uniq__order_info__order_no` |

长度超限时的缩写规则与对照表，见各数据库专项规范。

## 7. 保留字规避

表名/字段名若命中数据库保留关键字，必须按对应数据库专项规范的策略规避（如 PostgreSQL 的 `_info` 后缀），彻底避免 SQL 中必须加引号的麻烦。

## 8. 各数据库专项规范

| 数据库 | 文档 |
| :--- | :--- |
| PostgreSQL | [postgresql/naming.md](postgresql/naming.md) |
