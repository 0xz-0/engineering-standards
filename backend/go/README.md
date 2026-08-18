# Go 后端服务开发规范

Go 微服务开发相关的规范集合。

## 文档索引

| 文档 | 说明 | 状态 |
| --- | --- | --- |
| [项目结构规范](project-layout.md) | 目录结构、分层约定、包命名 | 📝 待编写 |
| [编码风格规范](coding-style.md) | 命名、代码组织、lint 要求 | 📝 待编写 |
| [错误处理规范](error-handling.md) | 错误定义、包装、传递与响应码约定 | 📝 待编写 |
| [日志规范](logging.md) | 日志级别、字段约定、trace 关联 | 📝 待编写 |
| [配置管理规范](configuration.md) | 配置来源、环境区分、敏感信息处理 | 📝 待编写 |

## 适用范围

- 所有 Go 编写的后端微服务；
- BFF 层如使用 Node.js / TypeScript，请参见 [`bff/`](../../bff/README.md) 目录。
