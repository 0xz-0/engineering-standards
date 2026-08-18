# Engineering Standards

团队工程规范仓库，沉淀数据库、后端服务、BFF 等各领域的命名与设计规范，作为日常开发、Code Review 和技术决策的统一依据。

## 规范目录

| 领域 | 目录 | 说明 |
| --- | --- | --- |
| 通用约定 | [`general/`](general/README.md) | 跨领域通用规范（版本管理、文档写作、Code Review 等） |
| 数据库 | [`database/`](database/README.md) | 数据库命名规范、表设计规范、索引规范、SQL 编写规范等 |
| 后端服务 | [`backend/`](backend/README.md) | 各语言后端服务开发规范，目前以 Go 为主 |
| BFF 服务 | [`bff/`](bff/README.md) | BFF 层服务开发规范（Node.js / TypeScript） |

## 文档约定

- 文档使用 Markdown 编写，正文以**中文为主**，技术名词保留英文（如 `snake_case`、Repository、DTO）。
- 每篇规范文档需遵循 [`templates/standard-template.md`](templates/standard-template.md) 的结构。
- 规范分为三个约束力级别，文档开头必须标注：
  - **MUST**：强制，违反需在 Code Review 中整改；
  - **SHOULD**：建议，违反需说明理由；
  - **MAY**：可选，供参考。

## 如何新增一篇规范

1. 复制 [`templates/standard-template.md`](templates/standard-template.md) 到对应领域目录下，按 `kebab-case` 命名文件；
2. 填写各小节内容，标注约束力级别；
3. 在对应领域目录的 `README.md` 索引表中登记；
4. 提交 PR，由相关领域负责人评审合并。

## 如何修订规范

- 规范修订同样走 PR 评审流程；
- 重大变更（影响存量代码 / 需要迁移）需在 [`CHANGELOG.md`](CHANGELOG.md) 中记录；
- 鼓励在文档中补充真实案例（正例 / 反例），避免只有干巴巴的规则。

## 本地校验

仓库配置了 [markdownlint](.markdownlint.json)，提交前请确保文档格式通过检查：

```bash
npx markdownlint "**/*.md"
```

使用 VS Code 时可安装 `DavidAnson.vscode-markdownlint` 插件获得实时提示。
