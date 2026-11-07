# RAGFlow 从入门到精通 · 教程站点

RAGFlow（[infiniflow/ragflow](https://github.com/infiniflow/ragflow)）的系统性中文教程，基于 **v0.27.2** 源码编写，共 12 章，从零基础到源码级精通。

- **在线阅读**：<https://jigangbo.github.io/ragflow-tutorial/>
- **源码仓库**：以 git 子模块链接（见 `.gitmodules`），未 fork——教程与上游源码解耦，仅作引用。
- **配套深度解析**：`project-analysis/`（源码级架构解析 8 册，位于 ragflow 工作区）。

## 章节结构

| 难度 | 章节 |
|---|---|
| 🟢 入门 | 00 阅读指南 · 01 背景：零基础理解 RAG · 02 认识 RAGFlow · 03 环境搭建 · 04 知识库实战 · 05 对话与搜索 |
| 🟡 进阶 | 06 API 与 SDK · 07 Agent 工作流 · 08 高级特性 |
| 🔴 精通 | 09 架构解密 · 10 源码开发与贡献 · 11 运维调优与故障排查 |

## 本地预览

```bash
uvx --from mkdocs-material --with mkdocs-awesome-pages-plugin --with jieba mkdocs serve
```

## 部署

推送到 `main` 后由 [.github/workflows/tutorial-docs.yml](.github/workflows/tutorial-docs.yml) 自动构建并发布到 GitHub Pages（Actions 源）。

## 仓库说明

```
tutorial/            教程内容（mkdocs docs_dir）
mkdocs.yml           Material 主题站点配置
ragflow              上游源码仓库的子模块链接（gitlink，按需 git submodule update --init）
```
