# Changelog

记录规范的**重大变更**（新增规范、影响存量代码的规则调整、需要迁移的变更）。日常措辞修正无需记录。

格式遵循 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/)。

## [Unreleased]

### Added

- 仓库初始化：建立目录结构与各领域占位文档。
- 数据库规范体系：确立「通用层 + 数据库专项层」两层结构；
  - 通用层：架构红线（禁用 FK/CHECK/触发器/视图/UUID 主键/数字枚举）、命名规范（id 与 no/code 语义分层）、表设计规范（审计字段、TEXT 枚举、注释与冗余红线）、索引规范（8 条铁律）；
  - PostgreSQL 专项层：命名（保留字 `_info`、63 字节缩写表）、表设计（IDENTITY/TEXT/timestamptz、collation、序列命名）、索引（部分索引、CREATE INDEX CONCURRENTLY）、完整 DDL 示例。
