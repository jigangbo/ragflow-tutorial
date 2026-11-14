# 第 10 章 源码开发与贡献

> 难度：🔴 精通 · 建议 90 分钟 · 目标：搭好三栈开发环境，跑通测试，按仓库规约提交 PR。

## 1. 从源码跑起来（三栈并行）

```bash
# ① 依赖与第三方资源
uv sync --python 3.13 --all-extras
uv run python3 ragflow_deps/download_deps.py        # NLTK 数据/HF 模型/tika 等

# ② 基础设施（只起依赖，不起主服务）
docker compose -f docker/docker-compose-base.yml up -d
# Windows/Mac 需在 hosts 加：127.0.0.1 es01 infinity mysql minio redis sandbox-executor-manager

# ③ Python 后端（API :9380 + task_executor + admin）
source .venv/bin/activate && export PYTHONPATH=$(pwd)
bash docker/launch_backend_service.sh               # 参数: ragflow|task_executor|admin|data_sync|all

# ④ 前端（:9222，dev 代理到 9380）
cd web && npm install && npm run dev
```

Go 栈开发：`bash build.sh --go`（自动处理 C++ 静态库与原生依赖），Go API 跑在 9384；前端 `.env` 的 `API_PROXY_SCHEME` 切换代理目标。

## 2. Go 构建与测试（必须用 build.sh）

> ⚠️ 仓库规约（`AGENTS.md`）：**不要裸跑 `go test` / `go build`**——会漏 CGO 标志与原生静态库（office_oxide/pdfium-static/pdf_oxide）。

```bash
bash build.sh --go                          # 构建二进制（bin/ragflow_server, bin/ragflow-cli）
bash build.sh --test ./internal/dao/...     # unit 层（默认 go test ./... 必须自包含）
bash build.sh --test-integration ./...      # 需真实 MySQL/MinIO/ES/Infinity
bash build.sh --test-e2e                    # 全管道
bash build.sh --test-all                    # integration + e2e（绝不含 manual）
```

| 层 | build tag | 默认跑？ | 允许依赖 |
|---|---|---|---|
| unit | 无 | ✅ | 内存 SQLite / miniredis / httptest；禁外部服务 |
| integration | `integration` | ❌ | 单组件 × 真实服务 |
| e2e | `e2e` | ❌ | ingest→index→retrieve 全管道 |
| manual | `manual` | ❌ **永不进 CI** | deepdoc render/parity/bench，仅本地 opt-in |
| native | `cgo`/`!cgo`（正交） | CGO_ENABLED=1 自动满足 | 原生静态库，可与层级组合 |

## 3. Python / 前端测试

```bash
uv run pytest                              # 全套（test/ 目录）
uv run pytest test/testcases/test_sdk_api  # 定向（推荐先跑窄的）
ruff check && ruff format                  # Lint（line-length 200）

cd web && npm run lint && npm run type-check && npm run test   # Jest 单测
# E2E（Python Playwright，需先起完整服务）：
BASE_URL=http://localhost:9222 E2E_ADMIN_EMAIL=admin@ragflow.io \
  E2E_ADMIN_PASSWORD=admin uv run pytest -q test/playwright
```

## 4. 代码地图（改哪去哪）

| 需求 | 触点 |
|---|---|
| 新 REST 端点（Python） | `api/apps/restful_apis/*_api.py`（蓝图自动注册，前缀 `/api/v1`） |
| 新业务表字段 | **两侧同步**：`api/db/db_models.py` + `internal/entity/`（索引按列集合互认，勿改名"修复"） |
| 新文档解析/分块策略 | `rag/app/` 新 chunker，注册进 `rag/svr/task_executor.py::FACTORY` |
| 新模型厂商 | `rag/llm/chat_model.py`（或 LiteLLMBase 前缀），同步 `docs/references/supported_models.mdx` |
| 新 Agent 组件 | `agent/component/`（自动扫描注册；Go 侧对齐 `internal/agent/component/`） |
| 新工具 | `agent/tools/`（ToolBase：`meta` 即 OpenAI function schema） |
| 前端页面 | `web/src/pages/`，遵循 `web/AGENTS.md`（双后端变体用 `<BackendVariant>`，禁直接读 `API_PROXY_SCHEME`） |

## 5. 仓库工作规约（AGENTS.md 精要）

- **旧代码是负债不是兼容目标**：优先删除 shim/deprecated 分支/双轨迁移注释，收敛到单一路径；
- 改前先读"真正拥有该行为的最近代码路径"；改动保持小而局部；
- 注释描述当前运行路径，不写迁移历史；不留"以后再移"的注释；
- 数据库：共享唯一索引必须两侧同时声明；Go 单列唯一用命名 `uniqueIndex:`（裸 `unique` 会触发幽灵删索引，见 `namedIndexMigrator`）；
- 维护者接管社区 PR 重写历史时，必须保留原作者并以 `Co-authored-by:` 添加维护者。

## 6. 提 PR 流程

1. Fork → 分支 → 小步提交（提交信息遵循现有风格，如 `fix(dataset): ...`）；
2. 跑窄验证：改动 Python → 定向 pytest + ruff；改 Go → `build.sh --test ./pkg/...`；改前端 → 触碰包的 lint/type-check/test；
3. 签署 DCO（`git commit -s`，Apache-2.0 项目要求）；
4. PR 描述写清动机/改动点/验证方式，关联 issue。

> 深读：各子系统代码导览见 [project-analysis/ 对应分册；Go 测试分层完整表见 [06-go-backend.md 第 10 节。
