# 阅读指南

> 本教程基于 RAGFlow **v0.27.2** 源码与官方文档编写，所有操作命令均可在仓库中找到出处。

## 三条阅读路径

### 路径 A：快速上手（约 2 小时）

目标：**今天就让 RAGFlow 跑起来并回答第一个问题**。

1. [01 背景知识](01-background.md) —— 只读第 1、2 节，建立 RAG 基本概念（15 分钟）
2. [03 环境搭建与首次运行](03-setup.md) —— Docker 一键启动，注册账号，配置模型（40 分钟）
3. [04 知识库实战](04-knowledge-base.md) —— 创建知识库、上传文档、解析、检索测试（40 分钟）
4. [05 对话助手与搜索应用](05-chat-and-search.md) —— 建第一个 Chat 助手并提问（25 分钟）

### 路径 B：完整学习（按顺序读完全部章节）

```
01 背景知识 → 02 认识RAGFlow → 03 环境搭建 → 04 知识库实战
→ 05 对话与搜索 → 06 API与SDK → 07 Agent工作流 → 08 高级特性
→ 09 架构解密 → 10 源码开发与贡献 → 11 运维调优与故障排查
```

每章末尾附"进阶线索"，指向源码位置与深度解析文档（`project-analysis/` 系列），适合想读源码的读者。

### 路径 C：专题查阅

| 你想做的事 | 直接读 |
|---|---|
| 用代码批量管理知识库/文档 | [06 API 与 SDK 编程](06-api-and-sdk.md) |
| 搭建自动化工作流/多智能体 | [07 Agent 工作流实战](07-agent-workflow.md) |
| 接入 Notion/飞书/数据库等外部数据 | [08 高级特性](08-advanced-features.md) 第 1 节 |
| 理解检索为什么这样排序/打引用 | [09 架构解密](09-architecture.md) |
| 二次开发、改源码、提 PR | [10 源码开发与贡献](10-development.md) |
| 服务变慢/起不来/检索不准 | [11 运维调优与故障排查](11-operations.md) |

## 难度标注

| 级别 | 章节 | 预备知识 |
|---|---|---|
| 🟢 入门 | 01–05 | 会用 Docker、浏览器；无需编程 |
| 🟡 进阶 | 06–08 | Python 基础；了解 HTTP API |
| 🔴 精通 | 09–11 | Python + Go（09/10 前半）；Linux 运维（11） |

## 配套资料

| 资料 | 位置 | 用途 |
|---|---|---|
| 官方 HTTP API 参考 | `docs/references/http_api_reference.md` | 第 06 章的完整端点字典 |
| 官方 Python API 参考 | `docs/references/python_api_reference.md` | SDK 每个方法的签名 |
| 深度架构解析（8 册） | [`project-analysis/` | 09/10/11 章的展开阅读 |
| SDK / HTTP 示例脚本 | `example/sdk/`、`example/http/` | 第 06 章的可运行版本 |
| 本仓库操作规约 | `AGENTS.md`、`web/AGENTS.md` | 第 10 章贡献规范 |
