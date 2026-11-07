# 对话助手与搜索应用

> 难度：🟢 入门 · 建议 25 分钟 · 目标：建好第一个 Chat 助手，理解检索参数与引用，会分享和嵌入。

## 1. 创建 Chat 助手

顶栏 **Chat** → **Create an Assistant**。关键配置分三组：

**① 助手设置**

| 配置 | 说明 |
|---|---|
| Name / Description | 名称与描述 |
| Empty response | 检索不到内容时的固定回复（防幻觉利器，如"知识库中没有相关内容"） |
| Show quote | 是否展示引用 |
| Cross-language | 允许跨语言检索（中文库回答英文提问） |

**② Prompt（System Prompt）**

默认 prompt 已包含"基于引用材料作答"的约束。可用变量：`{knowledge}`（检索到的块）、`{history}`（对话历史）。可自定义角色人设，但**不要删掉"基于知识库内容回答"的核心约束**。

**③ 检索设置（与检索测试页同源）**

| 参数 | 建议值 | 作用 |
|---|---|---|
| Top N | 6 | 送入 LLM 的块数 |
| Similarity threshold | 0.2 | 低于阈值的块丢弃 |
| Keywords similarity weight | 0.3 | 0=纯语义，1=纯关键词 |
| Rerank model | 可选 | 配置后 Top N 候选重排，推荐 |
| Knowledge base(s) | 多选 | 可同时挂多个知识库 |

## 2. 对话与引用

对话页提问后，答案中的 `[ID:1]` 角标点击可展开**原文块与截图**——这是 RAGFlow 的核心卖点。下方还有"下一问建议"。

**多轮对话**：默认携带对话历史，问题里的"它/上面说的"会被 LLM 改写成独立问题再检索（full question 改写，源码 `rag/prompts/full_question_prompt.md`）。

## 3. 分享与嵌入

| 方式 | 入口 | 用途 |
|---|---|---|
| 分享链接 | Chat 卡片 → Share | 免登录聊天页 |
| iframe 嵌入 | Chat 卡片 → Embed | 嵌到官网/工单系统 |
| Beta API | 管理界面 | 对外服务用专用 token |

## 4. 搜索应用（Search App）

如果只需要"检索"不需要"生成答案"（如企业搜索），用 **Search** 页创建搜索应用：可配置检索范围、展示脑图、相关推荐，同样可分享。适合站内搜索、资料检索门户场景。

## 5. 用 OpenAI 兼容端点接入现有应用

RAGFlow 暴露 OpenAI 格式的对话接口，现有任何支持自定义 OpenAI base_url 的客户端都能直连：

```python
from openai import OpenAI
client = OpenAI(
    base_url="http://localhost/api/v1/chats/<chat_id>/completions",  # RAGFlow 兼容端点
    api_key="ragflow-xxxxxxxx",                                       # RAGFlow API Key
)
resp = client.chat.completions.create(
    model="", messages=[{"role": "user", "content": "RAGFlow 是什么？"}]
)
print(resp.choices[0].message.content)
```

另有 Dify 兼容检索端点 `/api/v1/dify/retrieval`，方便从 Dify 迁移。

## 6. 音频能力

Chat 配置里可开启 TTS（需配 TTS 模型）；对话页支持语音播放。上传音频文件到知识库则走 ASR 转写分块。

> 进阶线索：对话请求的完整服务端旅程（SSE 事件、引用生成算法）见 [project-analysis/02-python-api-backend.md 第 2 节与 [03-rag-core.md 第 5 节。
