# RAGFlow 从入门到精通

> 系统性教程 · 基于 **v0.27.2** 源码 · 三篇 11 章 · 难度从零基础到源码级。

**RAGFlow** 是开源 RAG（检索增强生成）引擎 + Agent 平台：用深度文档理解（OCR/版面/表格识别）把各类文档变成可溯源的知识库，以混合检索 + 有据引用回答问题，并提供可视化画布编排 Agent 工作流。

## 整体架构总览

```
 ┌──────────────────────────── 用户侧 ────────────────────────────┐
 │  Web 浏览器      Python SDK / REST 调用方      IM 渠道 ×8       │
 │                (Telegram/Discord/飞书/钉钉/企微/LINE/QQ/WhatsApp)│
 └──────────────┬──────────────────┬──────────────────┬───────────┘
                ↓                  ↓                  ↓
 ┌────────── nginx :80 ───────────────────────────────────────────┐
 │  / → React 前端(Vite)   /api,/v1 → 按 API_PROXY_SCHEME 分流     │
 └───────┬───────────────────────────────────────┬────────────────┘
         ↓                                       ↓
 ┌───────────────┐  API_PROXY_SCHEME   ┌───────────────┐
 │ Python 栈 9380│◄────python|go|hybrid────►│ Go 栈 9384   │
 │ Quart+Peewee  │                     │ gin+GORM      │
 └───┬───────┬───┘                     └───┬───────┬───┘
     │       │                             │       │
 Redis     MySQL                       NATS JS    MySQL      ← 两侧共写同一 schema
 Stream  (39 张表)                     (Go 队列)  (共享)
     ↓                                     ↓
 ┌────────────────┐                ┌────────────────┐
 │ task_executor  │                │ Go ingestor    │   解析执行器
 │ rag/svr ×N     │                │ pdf/docx/txt   │
 └───┬────────────┘                └───┬────────────┘
     ↓                                 ↓
   DeepDoc 视觉推理 (ONNX: OCR/版面/TSR)  +  LLM/Embedding/Rerank 工厂 (33+ 厂商)
     ↓
 ┌──────────────────────────┐   ┌──────────────┐   ┌──────────────┐
 │ Elasticsearch/Infinity/… │   │ MinIO / S3   │   │ Admin 9381/3 │
 │ 全文+向量 (ragflow_{租户}) │   │ 文件/图片     │   │ 管理/监控     │
 └──────────────────────────┘   └──────────────┘   └──────────────┘
```

## 主工作流：文档摄取 → 检索问答

```
 【摄取流水线】
 (1) 上传文档 ──→ (2) 存 MinIO ──→ (3) 触发解析，任务切片入 Redis Stream
      ↓
 (4) task_executor 消费：DeepDoc 解析 → (5) chunker 分块（15 种模板）
      ↓
 (6) 增强处理：关键词/问题提案/元数据/标签 → (7) Embedding 向量化
      ↓
 (8) 写入检索引擎（父块+子块） ──→ (9) 进度聚合回 document ──→ (10) 前端轮询展示

 【检索问答】
 (1) 用户提问 ──→ (2) 多轮指代消解/查询改写
      ↓
 (3) 混合检索：全文查询(字段加权) + 向量检索 ──融合──→ (4) rerank 重排
      ↓
 (5) 阈值过滤取 Top N ──→ (6) LLM 生成 + [ID:n] 引用标注 ──→ (7) SSE 流式回包
      ↓
 (8) 点引用回看原文截图（DeepDoc 记录了每块的页面坐标）
```

## 代码-流程映射表

| 流程步骤 | 源码位置 | 对应章节 |
|---|---|---|
| 上传/解析入口 | `api/apps/restful_apis/document_api.py`、`api/db/services/task_service.py` | [第 4 章](part1-basics/04-knowledge-base.md) |
| 任务队列 | Redis Stream（`common/settings.py`、`rag/utils/redis_conn.py`） | [第 9 章](part3-mastery/09-architecture.md) |
| 执行器 | `rag/svr/task_executor.py`（Go：`internal/ingestion/`） | [第 9 章](part3-mastery/09-architecture.md) |
| 文档理解 | `deepdoc/parser/pdf_parser.py`、`deepdoc/vision/` | [第 4 章](part1-basics/04-knowledge-base.md) |
| 分块模板 | `rag/app/naive.py` 等 15 个（`FACTORY` 注册） | [第 4 章](part1-basics/04-knowledge-base.md) |
| 向量化 | `rag/llm/embedding_model.py`、`rag/svr/task_executor.py::embedding` | [第 9 章](part3-mastery/09-architecture.md) |
| 索引写入 | `rag/utils/es_conn.py`、`infinity_conn.py` 等 | [第 9 章](part3-mastery/09-architecture.md) |
| 混合检索 | `rag/nlp/query.py`（全文）+ `rag/nlp/search.py`（Dealer 融合） | [第 4 章](part1-basics/04-knowledge-base.md)/[第 9 章](part3-mastery/09-architecture.md) |
| 引用标注 | `rag/nlp/search.py::insert_citations` + `rag/prompts/citation_plus.md` | [第 5 章](part1-basics/05-chat-and-search.md) |
| 对话入口 | `api/apps/restful_apis/chat_api.py`、`api/db/services/dialog_service.py` | [第 5 章](part1-basics/05-chat-and-search.md) |
| Agent 引擎 | `agent/canvas.py`、`agent/component/`、`agent/tools/` | [第 7 章](part2-advanced/07-agent-workflow.md) |
| GraphRAG | `rag/graphrag/` | [第 8 章](part2-advanced/08-advanced-features.md) |
| 数据源同步 | `common/data_source/`、`rag/svr/sync_data_source.py` | [第 8 章](part2-advanced/08-advanced-features.md) |
| IM 渠道 | `api/channels/`（Go：`internal/channels/`） | [第 8 章](part2-advanced/08-advanced-features.md) |
| 前端画布 | `web/src/pages/agent/canvas/`（React Flow 12） | [第 7 章](part2-advanced/07-agent-workflow.md) |
| 部署编排 | `docker/entrypoint.sh`、`docker-compose*.yml`、`build.sh` | [第 3 章](part1-basics/03-setup.md)/[第 11 章](part3-mastery/11-operations.md) |

## 章节索引

### 第一篇 入门篇：从零跑通 RAGFlow

| 章 | 标题 | 一句话 |
|---|---|---|
| [阅读指南](00-reading-guide.md) | 三条阅读路径 | 快速上手 / 完整学习 / 专题查阅 |
| [第 1 章](part1-basics/01-background.md) | 背景：零基础理解 RAG | RAG 是什么、核心概念词汇表 |
| [第 2 章](part1-basics/02-meet-ragflow.md) | 认识 RAGFlow | 能力清单、界面导览、服务组成 |
| [第 3 章](part1-basics/03-setup.md) | 环境搭建与首次运行 | Docker 启动、配模型、常见坑 |
| [第 4 章](part1-basics/04-knowledge-base.md) | 知识库实战 | 上传解析、模板选择、检索调参 |
| [第 5 章](part1-basics/05-chat-and-search.md) | 对话助手与搜索应用 | Chat 配置、引用、分享嵌入、OpenAI 兼容 |

### 第二篇 进阶篇：API、Agent 与高级特性

| 章 | 标题 | 一句话 |
|---|---|---|
| [第 6 章](part2-advanced/06-api-and-sdk.md) | API 与 SDK 编程 | REST 速览、SDK 全流程代码 |
| [第 7 章](part2-advanced/07-agent-workflow.md) | Agent 工作流实战 | 模板、画布、组件、工具、沙箱、Webhook |
| [第 8 章](part2-advanced/08-advanced-features.md) | 高级特性 | 连接器、GraphRAG、RAPTOR、记忆、渠道 |

### 第三篇 精通篇：架构、源码与运维

| 章 | 标题 | 一句话 |
|---|---|---|
| [第 9 章](part3-mastery/09-architecture.md) | 架构解密 | 双后端、三条数据流、目录映射 |
| [第 10 章](part3-mastery/10-development.md) | 源码开发与贡献 | 三栈环境、build.sh、测试分层、规约 |
| [第 11 章](part3-mastery/11-operations.md) | 运维调优与故障排查 | 端口表、安全加固、调参、排障表 |

## 快速开始

```bash
git clone https://github.com/infiniflow/ragflow.git && cd ragflow
git checkout v0.27.2
cd docker && docker compose -f docker-compose.yml up -d
# 打开 http://localhost，注册后在 Model Providers 配置 LLM API Key
```

然后按 [阅读指南](00-reading-guide.md) 的路径 A 走。
