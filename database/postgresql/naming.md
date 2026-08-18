# PostgreSQL 命名规范

> **约束力**：MUST
> **适用范围**：PostgreSQL 业务库
> **说明**：通用命名原则（snake_case、单数表名、字段语义分层、后缀语义、逻辑外键、关联表、索引前缀）见[数据库命名规范](../naming.md)，本文仅约定 PostgreSQL 的落地差异。

## 1. 保留字规避策略（方案A）

若表名是 **PostgreSQL 保留关键字（Reserved Keyword）**，则**必须**追加后缀 `_info`，彻底规避 SQL 中必须加双引号的麻烦。

**判定依据**：以 [PostgreSQL 官方关键字表](https://www.postgresql.org/docs/current/sql-keywords-appendix.html) 中 **reserved** 级别为准，不凭感觉扩张清单。

**强制追加 `_info` 的高危词清单（至少包含以下）**：

`order` → `order_info` | `user` → `user_info` | `group` → `group_info` | `table` → `table_info` | `index` → `index_info` | `trigger` → `trigger_info` | `function` → `function_info` | `window` → `window_info` | `select` → `select_info` | `where` → `where_info`

> 注：`partition`、`procedure` 等在 PG 中为**非保留关键字**，可作表名/列名，不强制加后缀；但为统一心智，建议同业务域内风格保持一致。

**不受影响的其他表**：`product`、`category`、`inventory`、`address`、`payment` 等非保留字，**保持单数简洁形式**，不加后缀。

## 2. 索引名长度限制与缩写规则

- **硬上限**：PostgreSQL 标识符最大长度为 **63 字节**。
- **触发阈值**：生成的索引名长度 **> 60 字节** 时，必须启动缩写（当前全 ASCII 命名下"字符数=字节数"，统一按字节表述防止未来引入非 ASCII 缩写时误判）。
- **缩写优先级**：优先缩写**字段名**，其次缩写**表名**。
- **固定缩写对照表**：

| 原词 | 缩写 | 原词 | 缩写 |
| :--- | :--- | :--- | :--- |
| `transaction` | `txn` | `notification` | `notif` |
| `information` | `info` | `middleware` | `mw` |
| `description` | `desc` | `timestamp`（作后缀） | `ts` |
| `configuration` | `cfg` | `quantity` | `qty` |
| `address` | `addr` | `account` | `acct` |
| `number` | `num` | `message` | `msg` |
| `password` | `pwd` | `amount` | `amt` |
| `business` | `biz` | `category` | `cat` |
| `document` | `doc` | `product` | `prod` |
| `count` | `cnt` | `image` | `img` |
| `organization` | `org` | `position` | `pos` |

**严禁**对上表未收录的词自行缩写；确需新增缩写词时，先提 PR 补充本表再使用。

- **缩写示例**：`idx__transaction_log__transaction_id__notification_sent_at` → `idx__txn_log__txn_id__notif_sent_at`

## 3. 多业务域表名前缀

多业务域隔离通过**表名前缀**实现（如 `pay_order_info`、`msg_notification`），不划分额外 schema（见[表设计规范](table-design.md)）。
