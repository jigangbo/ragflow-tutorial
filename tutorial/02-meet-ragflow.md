# 认识 RAGFlow

> 难度：🟢 入门 · 建议 15 分钟 · 读完你将知道 RAGFlow 能做什么、界面上有什么。

## 1. RAGFlow 是什么

[RAGFlow](https://github.com/infiniflow/ragflow) 是开源的 **RAG 引擎 + Agent 平台**（Apache-2.0），官方口号："Deep document understanding"（深度文档理解）。它把"上传文档 → 自动理解 → 检索问答 → 可溯源引用"做成了一条完整流水线，并提供可视化工作流（Agent 画布）把 RAG 与工具调用组合成智能体。

- 版本（本教程基准）：**v0.27.2**
- 技术栈：Python 3.13（Quart API）/ Go 1.27（双后端）/ React 18 前端 / MySQL + Redis + MinIO + Elasticsearch（或 Infinity 等）
- 硬件门槛：CPU ≥4 核、内存 ≥16GB、磁盘 ≥50GB、Docker ≥24.0

## 2. 六大核心能力

| 能力 | 说明 | 在哪用 |
|---|---|---|
| 深度文档理解 | OCR + 版面分析 + 表格结构识别，支持 10 类版面元素；扫描件/双栏/表格均可解析 | 知识库解析配置 `layout_recognize` |
| 模板化分块 | 15 种切片模板（通用/问答/论文/法律/简历/表格/图片/音频…），策略可解释可配 | 知识库"切片方法" |
| 有据引用 | 检索与生成双侧标注 `[ID:n]`，点击引用回看原文位置截图 | 所有对话界面 |
| 混合检索 | 全文 + 向量加权融合，可调相似度阈值/关键词权重，可加 rerank | Chat 助手与检索测试页 |
| Agent 工作流 | 拖拽画布：LLM、检索、分类、循环、HTTP、代码沙箱、MCP 工具 | Agent 页面 |
| 生产化 | 多租户与团队、API Key、Python SDK、MCP Server、IM 渠道、30+ 数据源同步 | 系统设置/团队/数据源 |

## 3. 界面导览（登录后你会看到什么）

```
顶栏
├── 工作台 Home        四类应用入口卡片：知识库 / Chat / Agent / 搜索
├── Knowledge Base     知识库列表与详情（文档管理、解析状态、检索测试、配置）
├── Chat               聊天助手列表与对话页（可分享/嵌入）
├── Agent              Agent 画布与模板市场（可运行/调试/发布）
├── Search             可保存的搜索应用
├── File Management    个人文件空间（文件夹、上传、关联到知识库）
└── 头像菜单
    ├── 团队（多租户协作）
    ├── Model Providers（模型提供方，配 API Key）
    ├── API（生成 API Key）
    ├── Data Sources（外部数据源连接器）
    └── 版本信息
```

## 4. 它由哪些服务组成（运维视角）

Docker 启动后，`docker compose ps` 你会看到：

| 容器 | 作用 | 端口 |
|---|---|---|
| ragflow（主服务） | nginx + API + 任务执行器 + Admin | 80/443/9380/9381/9382 |
| mysql | 全部业务数据 | 3306 |
| redis (valkey) | 会话、任务队列、分布式锁 | 6379 |
| minio | 原始文件与图片对象存储 | 9000/9001 |
| es01（或 infinity/opensearch/oceanbase） | 全文 + 向量检索引擎 | 1200→9200 |

理解这张表，后面第 11 章排障时你就知道"哪个组件出问题该看哪个日志"。

> 进阶线索：进程角色与端口全景见 [project-analysis/01-architecture-overview.md 第 3 节。
