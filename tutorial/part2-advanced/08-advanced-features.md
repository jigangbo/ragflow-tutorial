# 第 8 章 高级特性

> 难度：🟡 进阶 · 建议 45 分钟 · 目标：掌握让检索质量与自动化程度更上一层的六项能力。

## 1. 外部数据源连接器（自动同步）

头像 → **Data Sources**：配置 Notion、Confluence、GitHub、GitLab、飞书 Wiki、Google Drive、Gmail、SharePoint、S3、Jira、Box、Dropbox、IMAP 邮箱、MySQL 等 **30+ 种来源**；再在知识库设置里绑定连接器，系统按计划**增量同步**（源码 `rag/svr/sync_data_source.py` 调度，Go 栈为 `internal/syncer`，默认 400 个并发 worker）。

典型用法：把 Confluence 空间映射成一个知识库，RAGFlow 定期拉新页并重新解析变更文档。

## 2. GraphRAG（知识图谱增强检索）

知识库 Configuration → **Extract knowledge graph** 开启。构建流程：LLM 分批抽取实体/关系 → 同名实体消歧合并 → Leiden 社区划分 → 生成社区报告 → node2vec 实体嵌入。适合**跨文档关联推理**（"A 公司和 B 公司共同投资过哪些项目"）。代价：解析时间与 token 消耗显著增加（建议只对高价值库开启）。解析完成后在知识库 **Graph 标签**查看图谱；对话时相关实体/社区会参与检索。

## 3. RAPTOR（摘要树）

Configuration → RAPTOR 开启：把分块逐层聚类 + LLM 摘要成树状结构，检索时"细节块"与"摘要节点"都可命中，**大幅改善全局性问题**（"这份年报的核心结论是什么"）。

## 4. 元数据过滤与标签

- **Metadata schema**：给文档定义字段（如 `部门/年份/密级`），上传时填写，检索时可作硬过滤（doc engine 侧下推，源码 `common/metadata_*_filter.py`）；
- **Tags**：用 Tag 模板建标签库，其他知识库的 chunk 自动打标，检索排序把标签匹配作为加分项（`tag_fea` 字段）。

## 5. Agent 记忆（Memory）

Agent 可跨会话记住用户偏好与事实：`memory` 表 + 独立检索（ES/Infinity 等多后端）。SDK 侧 `rag.create_memory/add_message/search_message`；Agent 工作流中有 MemorySave/Memory 节点。适合个性化助手（"记住我偏好简洁回答"）。

## 6. IM 渠道接入

系统设置 → **Chat Channels**：把 Agent 绑定到 **Telegram、Discord、飞书、钉钉、企业微信、LINE、QQ 机器人、WhatsApp** 八个渠道。架构是进程内网关 + reconcile 循环（改配置热生效）。微信个人号可用 `tools/chatgpt-on-wechat` 插件方案。

## 7. 知识编译产物（Wiki / 导航 / Skills）

高阶用法：知识库可把 chunk "编译"成结构化产品——**Wiki 页**（可增量更新）、**导航树（Navigation）**、**Skills**（结构化技能卡）。入口在知识库的 Compilation/Artifacts 页面，对应 API `/datasets/{id}/index` 系列任务。这是 RAGFlow 从"问答"走向"知识门户"的新方向（2026 年新增）。

## 8. 团队协作与多租户

- 头像 → **Team**：邀请成员，角色 owner/admin/member；知识库创建时选 Team 即共享；
- 每个租户的索引在 doc engine 里物理隔离（`ragflow_{tenant_id}`），数据不串；
- Admin 管理端（9381）可集中管理用户/角色/权限（RBAC）。

> 进阶线索：以上每项的源码位置与执行流程都在 [project-analysis/ 对应分册（GraphRAP→03 册、连接器→02/06 册、渠道→02 册、记忆→02 册、知识编译→03/06 册）。
