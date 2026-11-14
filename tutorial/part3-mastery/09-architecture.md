# 第 9 章 架构解密

> 难度：🔴 精通 · 建议 60 分钟 · 目标：建立 RAGFlow 源码级的心智模型。本章是 [project-analysis/ 八册深度解析的导读浓缩版。

## 1. 一张图看全貌

```
                    ┌─────────────── nginx :80 ───────────────┐
                    │  /        → React 前端静态资源           │
                    │  /api,/v1  → API 服务器（按 scheme 分流） │
                    └───────┬──────────────────┬──────────────┘
                            ↓                  ↓
                 ┌──────────────────┐  ┌──────────────────┐
                 │ Python 栈 (9380) │  │ Go 栈 (9384)     │   ← API_PROXY_SCHEME 切换
                 │ Quart + Peewee   │  │ gin + GORM       │
                 └───┬──────────┬───┘  └───┬──────────┬───┘
                     │          │          │          │
             Redis Stream   MySQL      NATS JetStream  同一个 MySQL
                     ↓          ↑          ↓          ↑
            ┌────────────────┐  │  ┌────────────────┐
            │ task_executor  │──┘  │ Go ingestor    │   ← 解析执行器（各自消费）
            │ (rag/svr, ×N)  │     │ (pdf/docx/txt) │
            └───┬────────────┘     └───┬────────────┘
                ↓                      ↓
        DeepDoc 视觉推理（ONNX：OCR/版面/TSR） + LLM/Embedding 工厂（33+ 厂商）
                ↓
        Elasticsearch / Infinity / OpenSearch / OceanBase  ← 全文+向量索引（按租户分索引）
        MinIO/S3 ← 原始文件与图片        MySQL ← 39 张业务表（两侧共写同一 schema）
```

**最重要的架构事实：双语言后端**。`API_PROXY_SCHEME=python|go|hybrid` 决定 nginx 把请求分给谁；两侧共写同一 MySQL schema，靠"列集合"而非索引名互认。截至 v0.27.2：Python 是全功能主路径，Go 覆盖新 REST 面并只认领 pdf/docx/txt 的摄取。

## 2. 三条数据流（背下来就能定位 90% 问题）

**① 摄取流（上传→可检索）**

```
POST /datasets/{id}/documents/parse
→ 文档按页/行切片（PDF 默认 12 页一任务）→ task 表落库
→ Redis Stream XADD te.<优先级>.common
→ task_executor 消费：deepdoc 解析 → chunker 分块 → 增强（关键词/问题/元数据）
  → embedding（标题/正文双向量加权）→ 写入 doc engine（父块+子块）
→ 进度直写 task 表 → API 进程每 6 秒聚合回 document.progress → 前端 5 秒轮询
```

**② 检索流（提问→答案）**

```
提问 → query 改写（多轮指代消解）→ FulltextQueryer 生成全文查询（字段加权：
      important_kwd^30 > question^20 > title^10 > content^2）
→ 与向量检索 FusionExpr(weighted_sum) 融合 → 取 64 候选
→ rerank 模型（或 KNN 二次取分）→ 阈值过滤 → Top N
→ LLM 生成 → insert_citations() 句级相似度打 [ID:n]
```

**③ Agent 流（画布→执行）**

```
POST /agents/chat/completions → 加载 DSL 副本 → Canvas.run()
→ path 工作队列批调度（asyncio + 5 线程池）→ 组件输出 partial(流式生成器)
→ Message 节点消费 → SSE 事件流（workflow_started/node_started/message/…）
→ 每轮结束把整张 DSL 快照存 conversation.dsl（断点续跑的载体）
```

## 3. 关键目录 → 职责映射

| 目录 | 职责 | 教程对应 |
|---|---|---|
| `api/apps/restful_apis/` | 30 个 REST 模块 ≈320 路由（`/api/v1`） | 06 章 |
| `api/db/` | Peewee 39 张表 + CommonService CRUD 基类 | 09 章 |
| `rag/app/` | 15 种 chunker（`FACTORY` 注册） | 04 章 |
| `rag/svr/task_executor.py` | 摄取执行器（Redis Stream 消费者） | 09 章 |
| `rag/llm/` | 七类模型工厂（chat/embedding/rerank/…×33+ 厂商） | 03/04 章 |
| `rag/nlp/` | 分词、词权重、混合检索 `Dealer`、引用标注 | 09 章 |
| `rag/graphrag/` `rag/advanced_rag/` | 知识图谱 / agentic RAG / 知识编译 | 08 章 |
| `deepdoc/` | PDF 视觉解析 + OCR/版面/TSR ONNX 推理 | 04 章 |
| `agent/` | Canvas DSL 引擎、组件库、工具、沙箱 | 07 章 |
| `internal/` | Go 后端：router/ingestion/parser/engine/dao/agent/harness/syncer | 10 章 |
| `web/src/` | React 前端（Vite + React Flow 画布 + TanStack Query） | — |

## 4. 存储分工速查

| 存储 | 放什么 | 为什么 |
|---|---|---|
| MySQL | 39 张业务表（用户/知识库/文档/chunk 计数/对话/Agent DSL…） | 事务与关系 |
| ES/Infinity/… | chunk 全文 + 向量（索引名 `ragflow_{tenant_id}`） | 检索 |
| MinIO/S3 | 原始文件、缩略图、引用截图、沙箱产物 | 大对象 |
| Redis | 队列/锁/会话/secret/工具 trace/进度计数 | 低延迟共享状态 |

## 5. 值得记住的工程细节

- **引用可回溯**：DeepDoc 给每个 chunk 记录页面坐标（`position_int`），前端点引用即裁剪原位截图；
- **任务幂等**：解析任务带 xxhash digest，配置未变可复用旧 chunk；`_prune_deleted_chunks` 在检索侧兜底删除不一致；
- **内存碎片治理**：executor 达到 `MAX_TASKS_PER_WORKER` 后自杀重启，规避 ONNX Runtime 碎片；
- **LLM 工厂双轨**：自研 OpenAI 兼容 Base + LiteLLMBase（33 家），厂商特判集中在 `_apply_model_family_policies`；
- **沙箱安全**：gVisor + seccomp + 默认断网 + 输出类型契约校验。

> 深读路线：先读 [project-analysis/01-architecture-overview.md，再按兴趣选 02–08 册。
