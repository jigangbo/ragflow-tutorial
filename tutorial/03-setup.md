# 环境搭建与首次运行

> 难度：🟢 入门 · 建议 40 分钟 · 目标：Docker 启动 RAGFlow，注册账号，配置好一个 LLM。

## 1. 前置条件

| 要求 | 最低值 | 说明 |
|---|---|---|
| CPU / 内存 / 磁盘 | ≥4 核 / ≥16GB / ≥50GB | 内存含 Docker 分配 |
| Docker | ≥24.0.0 | Docker Desktop 或原生 |
| Docker Compose | ≥v2.26.1 | `docker compose version` 验证 |
| Linux 内核参数 | `vm.max_map_count ≥ 262144` | 仅 Linux/WSL2 需要，Elasticsearch 要求 |

设置内核参数（Linux/WSL2，重启失效需写入 /etc/sysctl.conf）：

```bash
sudo sysctl -w vm.max_map_count=262144
```

> Windows + Docker Desktop：在 WSL2 发行版内执行上面的 sysctl 即可。macOS 无需此步。
> ⚠️ 官方暂无 ARM64 镜像（如 Apple Silicon 原生），ARM 需自行构建。

## 2. 启动（标准流程）

```bash
git clone https://github.com/infiniflow/ragflow.git
cd ragflow
git checkout v0.27.2        # 重要：锁定版本，保证 entrypoint.sh 与镜像一致
cd docker

# CPU 版（默认）
docker compose -f docker-compose.yml up -d
```

GPU 加速解析（可选）：编辑 `docker/.env` 把 `DEVICE` 改为 `gpu` 后重新 up。

镜像说明：v0.22.0 起只发布 **slim 版**（≈2GB，不含内置 embedding 模型），模型全部走外部 API 或 TEI 容器。

## 3. 观察启动状态

```bash
docker logs -f docker-ragflow-cpu-1
# 看到 ASCII 大字 "RAGFlow" banner 即启动成功，按 Ctrl+C 退出日志（不影响服务）
```

启动失败先看这一行日志定位组件：数据库连不上 / ES 超时 / 端口占用是三大常见原因（详见 [11 运维与排障](11-operations.md)）。

## 4. 首次登录

1. 浏览器打开 `http://服务器IP`（nginx 监听 80；本机即 http://localhost）；
2. Sign up 注册账号（第一个注册用户自动成为租户 owner）；
3. 登录后进入工作台。

## 5. 配置模型（必做，否则无法解析/对话）

RAGFlow 不内置任何 LLM。你需要至少配置：

- 一个 **Embedding 模型**（解析文档必须）
- 一个 **Chat 模型**（对话必须）

**方式 A：界面配置（推荐）**

右上角头像 → **Model Providers** → 在列表中找到你的厂商（OpenAI / DeepSeek / 通义千问 / Ollama …）→ **Add the model** 填入 API Key → 回到该厂商卡片，把要用的模型 **Set as default**（分别设置 chat / embedding / rerank 等默认模型）。

**方式 B：配置文件预置（无人值守部署）**

编辑 `docker/service_conf.yaml.template` 的 `user_default_llm` 段：

```yaml
user_default_llm:
  factory: dashscope          # 厂商标识（见 conf/llm_factories.json 的 name）
  api_key: sk-xxxx
  base_url: ''                # 可选
  default_models:
    chat_model: 'qwen-plus'
    embedding_model: 'text-embedding-v3'
```

然后 `docker compose restart`。

**本地模型（无外网）**：用 Ollama —— Model Providers 选 Ollama，地址填 `http://host.docker.internal:11434`；或启用内置 TEI embedding 容器（`.env` 中 profile 加 `,tei-cpu`）。

## 6. 验证安装

进入任一知识库 → **检索测试（Retrieval Testing）** 页能出分，说明 embedding 链路通；任意对话能流式回复，说明 chat 链路通。

## 7. 常用运维命令

```bash
docker compose ps                    # 查看各容器状态
docker compose logs -f ragflow       # 跟踪主服务日志
docker compose stop / start          # 停 / 起
docker compose down                  # 删除容器（保留数据卷）
docker compose down -v               # ⚠️ 连数据卷一起删（全部知识库丢失）
```

## 8. 进阶启动变体（了解即可）

| 变体 | 做法 | 适用 |
|---|---|---|
| 切换检索引擎 | `.env` 中 `DOC_ENGINE=infinity|opensearch|oceanbase` 后 `docker compose down -v && up` | 想换引擎（切换需清空数据） |
| Go 后端 | 使用 `docker-compose-go.yml`（`.env-go`），Go API 端口 9384 | 体验 Go 栈 |
| 开 MCP Server | 主服务 command 加 `--enable-mcpserver`（默认 9382） | 给 Claude 等 MCP 客户端用 |
| 代码执行沙箱 | profile 加 `,sandbox`，需 gVisor | Agent 里跑代码 |
| 从源码跑 | 见 [10 源码开发与贡献](10-development.md) | 二次开发 |

> 进阶线索：compose 服务矩阵与 profile 机制全解见 [project-analysis/08-deployment-and-ecosystem.md 第 1 节。
