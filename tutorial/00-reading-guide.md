# 阅读指南

> 本教程基于 RAGFlow **v0.27.2** 源码与官方文档编写，所有操作命令均可在仓库中找到出处。全站共三篇 11 章，章号全书连续。

## 三条阅读路径

### 路径 A：快速上手（约 2 小时）

目标：**今天就让 RAGFlow 跑起来并回答第一个问题**。

1. [第 1 章 背景：零基础理解 RAG](part1-basics/01-background.md) —— 只读第 1、2 节，建立 RAG 基本概念（15 分钟）
2. [第 3 章 环境搭建与首次运行](part1-basics/03-setup.md) —— Docker 一键启动，注册账号，配置模型（40 分钟）
3. [第 4 章 知识库实战](part1-basics/04-knowledge-base.md) —— 创建知识库、上传文档、解析、检索测试（40 分钟）
4. [第 5 章 对话助手与搜索应用](part1-basics/05-chat-and-search.md) —— 建第一个 Chat 助手并提问（25 分钟）

### 路径 B：完整学习（按顺序读完全部章节）

```
第一篇 入门篇
  第 1 章 背景知识 → 第 2 章 认识RAGFlow → 第 3 章 环境搭建
  → 第 4 章 知识库实战 → 第 5 章 对话与搜索
第二篇 进阶篇
  第 6 章 API与SDK → 第 7 章 Agent工作流 → 第 8 章 高级特性
第三篇 精通篇
  第 9 章 架构解密 → 第 10 章 源码开发 → 第 11 章 运维调优
```

每章末尾附"进阶线索"，指向源码位置与深度解析文档（ragflow 工作区 `project-analysis/` 系列），适合想读源码的读者。

### 路径 C：专题查阅

| 你想做的事 | 直接读 |
|---|---|
| 用代码批量管理知识库/文档 | [第 6 章 API 与 SDK 编程](part2-advanced/06-api-and-sdk.md) |
| 搭建自动化工作流/多智能体 | [第 7 章 Agent 工作流实战](part2-advanced/07-agent-workflow.md) |
| 接入 Notion/飞书/数据库等外部数据 | [第 8 章 高级特性](part2-advanced/08-advanced-features.md) 第 1 节 |
| 理解检索为什么这样排序/打引用 | [第 9 章 架构解密](part3-mastery/09-architecture.md) |
| 二次开发、改源码、提 PR | [第 10 章 源码开发与贡献](part3-mastery/10-development.md) |
| 服务变慢/起不来/检索不准 | [第 11 章 运维调优与故障排查](part3-mastery/11-operations.md) |

## 难度分级

| 级别 | 范围 | 预备知识 |
|---|---|---|
| 🟢 入门 | 第一篇（第 1–5 章） | 会用 Docker、浏览器；无需编程 |
| 🟡 进阶 | 第二篇（第 6–8 章） | Python 基础；了解 HTTP API |
| 🔴 精通 | 第三篇（第 9–11 章） | Python + Go（第 9/10 章）；Linux 运维（第 11 章） |

## 配套资料

| 资料 | 位置 | 用途 |
|---|---|---|
| 官方 HTTP API 参考 | ragflow 仓库 `docs/references/http_api_reference.md` | 第 6 章的完整端点字典 |
| 官方 Python API 参考 | ragflow 仓库 `docs/references/python_api_reference.md` | SDK 每个方法的签名 |
| 深度架构解析（8 册） | ragflow 工作区 `project-analysis/` 目录 | 第 9/10/11 章的展开阅读 |
| SDK / HTTP 示例脚本 | ragflow 仓库 `example/sdk/`、`example/http/` | 第 6 章的可运行版本 |
| 本仓库操作规约 | ragflow 仓库 `AGENTS.md`、`web/AGENTS.md` | 第 10 章贡献规范 |
