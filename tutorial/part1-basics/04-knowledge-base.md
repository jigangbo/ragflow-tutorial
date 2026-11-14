# 第 4 章 知识库实战

> 难度：🟢 入门 · 建议 40 分钟 · 目标：建好一个能被高质量检索的知识库，并看懂解析配置每一项。

## 1. 创建知识库

工作台 → **Knowledge Base** → **Create knowledge base**：填名称、描述、选择 **Embedding 模型**（⚠️ 知识库创建后 embedding 模型不可更改，不同库可用不同模型）→ 权限（Team / 仅自己）。

## 2. 上传文档

知识库详情 → **Documents** → 拖拽上传（单个文件最大由 `MAX_CONTENT_LENGTH` 控制，默认 1GB）。支持 PDF、Word、PPT、Excel、TXT、Markdown、HTML、EPUB、图片（扫描件）、音频（先 ASR）、EML 邮件等。

上传 ≠ 解析：文件只是存进对象存储，要点后面的 **Parse 按钮**（或勾选后批量 Parse）才开始切块和向量化。进度列实时刷新（前端 5 秒轮询），状态从 `RUNNING` 到 `DONE`/`FAIL`。

## 3. 选择切片模板（最重要的决定）

每个知识库或每个文档可指定 **Chunk method（切片方法）**，共 15 种。选错模板是"检索不准"的第一大原因：

| 模板 | 适用文档 | 切分逻辑 |
|---|---|---|
| **General（naive）** | 大多数文档，默认选择 | 按 delimiter + 块 token 数（默认 512）合并，支持父子块 |
| Q&A | FAQ 表格、问答对 | Excel 一行一问一答；PDF 按问题编号识别 |
| Paper | 学术论文（PDF） | 识别标题层级，摘要独立成块并加权 |
| Book | 长篇书籍 | 大块 + 中文标点分隔 |
| Laws | 法规条文 | 按条文层级合并，检索不切断条款 |
| Manual | 产品手册（PDF） | 段落 + 表格转自然语言 |
| Table | 结构化表格（xlsx/csv） | 一行一块，列头支持同义词语法 `性别/sex(male,female)` |
| Resume | 简历（PDF） | 结构化解析出近百字段（学历/工作经历…） |
| Presentation | PPT | 每页一块 + 页面缩略图 |
| Picture | 图片/扫描页 | OCR 提文，可叠加视觉 LLM 生成描述 |
| One | 短文件 | 整个文件一块 |
| Audio | 录音 | 先 ASR 转写再一块 |
| Email | .eml 邮件 | 头 + 正文 + 附件 |
| Tag | 标签语料 | 构建"内容-标签"库，供其他库打标排序 |

**经验法则**：拿不准就用 General；Excel 问答用 Q&A；法务/制度文件用 Laws；表格数据用 Table。

## 4. 看懂解析配置（General 模板为例）

知识库 **Configuration** 页：

| 配置项 | 默认 | 含义与调法 |
|---|---|---|
| Chunk token number | 512 | 块越大上下文越全但精度降；FAQ 类可调小 |
| delimiter | `\n` | 分句符，支持自定义 |
| **Layout recognition** | DeepDOC | PDF 解析后端：`DeepDOC`（本地视觉模型）/ `Naive`（纯文本，快但不识别版面）/ `MinerU`、`Docling`、`PaddleOCR` 等外部引擎 |
| Extract knowledge graph | 关 | 开启后构建 GraphRAG（见 [08 高级特性](../part2-advanced/08-advanced-features.md)） |
| RAPTOR | 关 | 构建摘要树，利于全局性问题 |
| Auto-keyword / Auto-question | 0 | LLM 为每块抽取 N 个关键词/生成 N 个问题，提升召回 |
| Extract metadata / TOC | 关 | LLM 抽元数据 / 生成目录 |

**解析失败排查三步**：① 看 Documents 列表的 `progress_msg`（悬停可见详细错误）；② embedding 模型是否配好、额度是否用完；③ 换 `Naive` 布局识别试一次，能过则说明是视觉模型环节的问题。

## 5. 查看 Chunk 与检索测试

- **Chunks 标签**：解析完成后逐块查看内容、关键词、位置；点缩略图可看原文截图（DeepDoc 记录了每块的页面坐标）。
- **Retrieval Testing 标签**：输入问题实时测试检索，**这是调参主战场**：
  - 相似度阈值（Similarity threshold）：太低引入噪声，太高召回不足，常用 0.2；
  - 关键词相似度权重（Keyword similarity weight）：0=纯向量，1=纯关键词，常用 0.3；
  - 开启 rerank 后 Top N 会重排，相关块更靠前。

## 6. 知识库级进阶操作

| 操作 | 入口 | 用途 |
|---|---|---|
| Knowledge Graph | 左侧 Graph 标签 | GraphRAG 抽取的实体关系图谱可视化 |
| Mindmap | 配置或对话页 | 文档脑图 |
| RAPTOR 树 | Configuration 开关 | 分层摘要树 |
| Metadata | Configuration → Metadata schema | 给文档定义元数据字段，检索时可过滤 |
| Tags | 配置页 | 跨库标签体系 |
| 团队共享 | 创建时选 Team | 多租户协作（成员角色 owner/admin/member） |

## 7. 检索不理想的调参清单（速查）

1. 分块不合适 → 换模板 / 调块大小 / 开 Auto-keyword；
2. 扫描件乱码 → 确认 Layout recognition 未选 Naive；
3. 关键术语搜不到 → 调高关键词权重 / 检查语言设置；
4. 答非所问但块正确 → 去 Chat 配置调 prompt（下一章）；
5. 大文档解析超慢 → 拆分上传，或换外部解析引擎。

> 进阶线索：15 种模板的源码实现与混合检索算法见 [project-analysis/03-rag-core.md。
