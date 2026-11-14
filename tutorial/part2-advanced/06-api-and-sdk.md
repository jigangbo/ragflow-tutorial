# 第 6 章 API 与 SDK 编程

> 难度：🟡 进阶 · 建议 45 分钟 · 目标：用 Python SDK / REST API 完成知识库全生命周期管理。

## 1. 获取 API Key

界面右上角头像 → **API** → **Create API Key**，得到形如 `ragflow-IzZmY1...` 的密钥。所有 REST 请求带 `Authorization: Bearer <API_KEY>`。

## 2. REST API 速览

Base URL：`http://<host>/api/v1`。完整字典见 `docs/references/http_api_reference.md`，高频端点：

| 端点 | 方法 | 用途 |
|---|---|---|
| `/datasets` | POST/GET/PUT/DELETE | 知识库 CRUD |
| `/datasets/{id}/documents` | POST/GET | 上传 / 列出文档 |
| `/datasets/{id}/documents/parse` | POST | 触发解析（body: `{"document_ids": [...]}`） |
| `/datasets/{id}/chunks` | GET/POST | 列出 / 手动添加 chunk |
| `/retrieval` | POST | 纯检索（不生成答案） |
| `/chats` | POST/GET | 助手 CRUD |
| `/chats/{chat_id}/sessions` | POST | 创建会话 |
| `/chats/{chat_id}/completions` | POST | 对话（SSE 流式） |
| `/agents/{id}/sessions` + `/agents/chat/completions` | POST | Agent 会话与执行 |
| `/system/healthz` | GET | 健康检查 |

curl 示例（完整可运行版本在 `example/http/`）：

```bash
# 创建知识库
curl -X POST http://localhost/api/v1/datasets \
  -H "Authorization: Bearer $API_KEY" -H "Content-Type: application/json" \
  -d '{"name": "product_docs", "embedding_model": "BAAI/bge-m3"}'

# 检索
curl -X POST http://localhost/api/v1/retrieval \
  -H "Authorization: Bearer $API_KEY" -H "Content-Type: application/json" \
  -d '{"question": "退货政策是什么", "dataset_ids": ["<dataset_id>"], "top_k": 5}'
```

## 3. Python SDK 全流程（可直接运行）

安装：`pip install ragflow-sdk`（版本与服务端保持一致，v0.27.2）。

```python
from ragflow_sdk import RAGFlow
import time

rag = RAGFlow(api_key="ragflow-xxxx", base_url="http://127.0.0.1")

# ① 创建知识库
dataset = rag.create_dataset(name="demo_kb", embedding_model="BAAI/bge-m3")

# ② 上传文档（内存内容或本地文件）
with open("manual.pdf", "rb") as f:
    docs = dataset.upload_documents([{"display_name": "manual.pdf", "blob": f.read()}])
doc = docs[0]

# ③ 触发解析并等待完成
dataset.async_parse_documents([doc.id])
while True:
    d = dataset.list_documents(id=doc.id)[0]
    if d.run == "DONE" and d.progress >= 1.0:
        break
    if d.run == "FAIL":
        raise RuntimeError(d.process_msg)
    print(f"解析进度: {d.progress:.0%}"); time.sleep(2)

# ④ 检索（不生成答案，拿原始块）
chunks = rag.retrieve(dataset_ids=[dataset.id], question="如何配置告警",
                      top_k=5, similarity_threshold=0.1, keyword=True)
for c in chunks:
    print(round(c.similarity, 4), c.document_name, c.content[:60])

# ⑤ 创建助手 + 会话 + 流式问答
assistant = rag.create_chat(name="manual_bot", dataset_ids=[dataset.id])
session = assistant.create_session()
for ans in session.ask("设备的保修期是多久？"):
    print(ans.content, end="", flush=True)   # 流式增量
print("\n引用:", ans.reference_list[:2])      # 引用块元数据

# ⑥ 精细操作 chunk（手动增改删）
chunk = doc.add_chunk(content="自定义补充内容", important_keywords=["保修"])
chunk.update({"content": "修改后的内容"})
```

运行级示例脚本：`example/sdk/{dataset,document,chunk,retrieval,chat_assistant}_example.py`。

## 4. SDK 能力地图

| 类 | 管什么 | 高频方法 |
|---|---|---|
| `RAGFlow` | 顶层 | create/delete_datasets, retrieve, create_chat, Agent CRUD, Memory API |
| `DataSet` | 知识库 | upload_documents, async_parse_documents, list_documents, get_auto_metadata |
| `Document` | 文档 | download, list_chunks, add_chunk, delete_chunks |
| `Chunk` | 分块 | update |
| `Chat` / `Agent` | 助手/智能体 | create_session, list_sessions |
| `Session` | 会话 | **ask（流式问答）** |
| `Memory` | 记忆体 | add_message, search_message |

## 5. 对话开发注意事项

- **流式**：`session.ask()` 返回生成器，逐段 yield；`ans.reference_list` 在结束时填充完整引用；
- **多租户**：API Key 绑定租户，密钥即身份，勿入前端代码；
- **限长**：Top N 块 + prompt 会占用上下文，块太多会被截断（`message_fit_in` 机制）；
- **幂等**：上传前可先 `list_documents(name=...)` 查重，避免重复解析。

> 进阶线索：REST 面全部 30 个模块 ≈320 条路由的分组清单见 [project-analysis/02-python-api-backend.md 第 2.4 节。
