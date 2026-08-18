# 数据库规范

数据库相关的命名、设计、SQL 编写等规范。

## 分层结构

- **通用层**：本目录下的规范文档，沉淀跨数据库的通用原则（"为什么"）；
- **数据库层**：`postgresql/`、`mysql/` 等子目录，存放具体数据库的落地细则（"怎么做"），只写与通用层的差异。

## 通用规范索引

| 文档 | 说明 | 状态 |
| --- | --- | --- |
| [架构红线规范](architecture.md) | 应用层与数据库的责任边界、禁用清单（FK/CHECK/触发器/视图等）、逻辑外键约定 | ✅ 已生效 |
| [命名规范](naming.md) | 命名风格、字段语义分层（id vs no/code）、后缀语义、关联表与索引命名 | ✅ 已生效 |
| [表设计规范](table-design.md) | 主键/金额/时间/枚举选型、审计字段、注释规范、冗余字段红线 | ✅ 已生效 |
| [索引规范](indexing.md) | 索引创建 8 条铁律、索引数量与写入成本 | ✅ 已生效 |
| [SQL 编写规范](sql-style.md) | SQL 书写格式、查询编写约定 | 📝 待编写 |

## 数据库专项规范索引

| 数据库 | 文档 | 说明 | 状态 |
| --- | --- | --- | --- |
| PostgreSQL | [命名规范](postgresql/naming.md) | 保留字 `_info` 清单、索引名 63 字节上限与缩写对照表 | ✅ 已生效 |
| PostgreSQL | [表设计规范](postgresql/table-design.md) | IDENTITY/TEXT/timestamptz 类型选型、collation、`now()` 语义、序列命名 | ✅ 已生效 |
| PostgreSQL | [索引规范](postgresql/indexing.md) | 部分索引、`CREATE INDEX CONCURRENTLY` 语法 | ✅ 已生效 |
| PostgreSQL | [DDL 完整示例](postgresql/ddl-example.md) | 符合全部规范的三张表示例（主表/明细表/关联表） | ✅ 已生效 |

## 适用范围

- 关系型数据库（PostgreSQL / MySQL 等）；
- 如引入其他存储（Redis、MongoDB 等），后续在本目录下补充对应规范。
