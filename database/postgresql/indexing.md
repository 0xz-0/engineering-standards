# PostgreSQL 索引规范

> **约束力**：MUST
> **适用范围**：PostgreSQL 业务库
> **说明**：通用索引设计原则（8 条铁律）见[数据库索引规范](../indexing.md)，本文仅给出 PostgreSQL 的落地语法。

## 1. 部分唯一索引（软删除场景，MUST）

含 `deleted_at` 的表，业务唯一键必须写成部分唯一索引：

```sql
CREATE UNIQUE INDEX uniq__order_info__order_no
    ON order_info(order_no) WHERE deleted_at IS NULL;
```

## 2. 部分索引（高频过滤条件）

对 `WHERE deleted_at IS NULL` 的常驻过滤条件，可把该条件做进索引谓词：

```sql
CREATE INDEX idx__order_info__user_id__created_at
    ON order_info(user_id, created_at) WHERE deleted_at IS NULL;
```

## 3. 纯关联表联合唯一索引（MUST）

```sql
CREATE UNIQUE INDEX uniq__user_role_relation__user_id__role_id
    ON user_role_relation(user_id, role_id);
-- 反向查询索引（前导列不同，不构成冗余）
CREATE INDEX idx__user_role_relation__role_id ON user_role_relation(role_id);
```

## 4. 生产环境并发建索引（MUST）

普通 `CREATE INDEX` 持有 `SHARE` 锁，会阻塞目标表写入，生产大表直接执行即为事故。生产环境一律使用：

```sql
CREATE INDEX CONCURRENTLY idx__order_info__user_id__created_at
    ON order_info(user_id, created_at);
```

**注意**：`CONCURRENTLY` **不能在事务块内执行**，使用 Flyway / Liquibase 等迁移工具时需关闭单迁移事务包裹（如 Flyway 对该脚本单独配置 `executeInTransaction=false`）。
