# Agent 工作流实战

> 难度：🟡 进阶 · 建议 60 分钟 · 目标：从模板创建第一个 Agent，看懂画布与 DSL，会加工具和调试。

## 1. 概念：Agent = 画布上的一张流程图

RAGFlow 的 Agent 是**可视化 DAG 工作流**：节点是组件（LLM、检索、分支、循环、代码……），边是执行顺序，LLM 在其中既能"生成内容"也能"选择走哪条边"。运行时由 Python 的 Canvas 引擎（`agent/canvas.py`）按 DSL 执行，支持流式输出到聊天界面。

三个入口：**Agent 列表 → 从模板创建 / 从空白创建 / 导入 DSL（JSON）**。

## 2. 从模板开始（推荐路径）

模板市场内置 19 个模板，建议按顺序玩三个：

| 模板 | 学到什么 |
|---|---|
| Web Search Assistant | 最简结构：Begin → Agent（带搜索工具）→ Message |
| Customer Feedback Dispatcher | **Categorize 分类路由**：一条输入分发给不同处理分支 |
| Data Analysis Beginner | **代码沙箱**：Agent 生成 Python，沙箱执行出图 |
| Deep Research | 多智能体：主 Agent 编排搜索专员/精读专员/综合者三个子 Agent |

点模板 → 自动复制为你的 Agent → **Run** 试跑。

## 3. 画布操作速成

- **加节点**：拖连接线上的 ➕，或从左侧组件面板拖入；
- **节点配置**：点节点右侧抽屉表单（每个组件有专属表单）；
- **全局变量**：Begin 节点里定义 `env.xxx`，运行时从用户输入注入；
- **调试**：Run 面板实时看每个节点的输入/输出/耗时（工具调用 trace 存在 Redis，10 分钟 TTL）；
- **版本**：Publish 发布为正式版；历史版本可回滚（`user_canvas_version` 表）。

## 4. 核心组件清单（按用途分类）

| 类别 | 组件 | 说明 |
|---|---|---|
| 入口/出口 | Begin、Message | Begin 支持三种模式：conversational（聊天）/ task（一次性）/ **Webhook（HTTP 触发）**；Message 可 Jinja2 渲染并支持文件下载 |
| LLM | LLM、Agent、Categorize | LLM 纯生成；**Agent = LLM + 工具循环**（ReAct 交给模型层 function calling）；Categorize 让 LLM 从预定义类别中选一个走对应分支 |
| 知识 | Retrieval（工具形态） | 混合检索你的知识库，结果进入引用池 |
| 流控 | Switch、Iteration、Loop、Variable Aggregator/Assigner | 规则分支 / 数组迭代子图 / while 循环 / 分支汇聚 |
| 集成 | Invoke（HTTP）、Code（沙箱）、ExeSQL、Email、Excel、DocGenerator | 外部调用 / Python·Node 执行 / 查库 / 发信 / 表格 / 生成文档 |
| 感知 | Browser | browser-use 驱动的网页浏览节点 |
| 数据 | DataOperations、ListOperations、StringTransform | 结构化数据处理 |

**变量引用语法**（全画布通用，必须掌握）：

```
{Agent:xxx@content}   引用某组件的输出
{sys.query}           系统变量：用户问题
{env.api_key}         全局环境变量
{item} / {index}      Iteration 循环体内的当前项/下标
```

## 5. 给 Agent 装工具

Agent 节点配置里 `Tools` 可挂：

- **内置工具**（约 30 个）：搜索（DuckDuckGo/Tavily/SearXNG/Google）、学术（arXiv/Scholar/PubMed/Wikipedia）、金融（Yahoo Finance/TuShare/AkShare/问财）、ExeSQL、Email、Crawler、GitHub 等；
- **MCP 服务器**：系统设置 → MCP 里注册外部 MCP server，Agent 即可调用其工具（RAGFlow 自身也对外暴露 MCP Server，含 `ragflow_retrieval` 等 3 个工具）；
- **子 Agent**：Agent 节点可作为另一个 Agent 的工具（多智能体）。

工具的元数据（OpenAI function schema）同时派生前端表单与 LLM 的工具描述——一处定义三处使用。

## 6. 代码沙箱

Code 组件把 Python/Node 代码发到独立沙箱容器执行（**gVisor 隔离 + seccomp + 默认断网**）。要点：

- 代码必须定义 `main(**kwargs)`，输入经参数注入、输出经类型契约校验；
- 产物文件（如 matplotlib 图片）自动回传并挂到对话下载；
- 自建部署时启动 sandbox profile（需 gVisor），或接 e2b/Aliyun/UCloud 等托管沙箱。

## 7. Webhook 触发与定时场景

Begin 节点选 **Webhook 模式**后，Agent 获得一个 HTTP 入口：`POST /api/v1/agents/<id>/webhook`，任何外部系统（CI、工单、IM 机器人）都能触发运行，payload 映射到表单变量。安全项：方法白名单、Body 大小限制、IP 白名单。

## 8. 调试心法

1. 先单独跑 Agent 节点（把上游先固定住）；
2. 看节点 `_ERROR` 输出——异常分支会显示在 trace 里；
3. 工具没被调用 → 检查工具 schema 与 prompt 是否冲突（让模型"必须调用工具"可在 prompt 明示）；
4. 循环不退出 → Loop 组件设 `maximum_loop_count` 兜底；
5. JSON 解析失败 → LLM 组件开启结构化输出（内置 json_repair 重试）。

> 进阶线索：Canvas 执行模型（批调度、partial 流式、断点续跑）源码级解析见 [project-analysis/05-agent-system.md；Go 移植运行时见 [06-go-backend.md 第 7 节。
