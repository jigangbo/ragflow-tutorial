# 第 11 章 运维调优与故障排查

> 难度：🔴 精通 · 定位：生产部署与日常运维手册。

## 1. 端口总表

| 端口 | 服务 | 端口 | 服务 |
|---|---|---|---|
| 80/443 | nginx（Web + 反代） | 9380 | Python API |
| 9381 | Python Admin | 9382 | MCP Server |
| 9383 / 9384 | Go Admin / Go API | 9385 | 沙箱执行器（仅 127.0.0.1） |
| 9390 | DeepDoc 独立推理（可选） | 9222 | 前端 dev / Playwright |
| 1200→9200 | Elasticsearch | 1201→9201 | OpenSearch |
| 23817/23820/5432 | Infinity | 2881 | OceanBase/SeekDB |
| 3306 | MySQL | 9000/9001 | MinIO |
| 6379 | Valkey/Kvrocks | 4222 | NATS |
| 6380 | TEI embedding | 16686 | Jaeger UI |

## 2. 安全加固清单（生产必做）

1. **改默认密码**：`docker/.env` 中 `ELASTIC_PASSWORD/MYSQL_PASSWORD/MINIO_PASSWORD`（默认全是 `infini_rag_flow`）；
2. **Admin 密码**：设 `ADMIN_DEFAULT_PASSWORD`（默认 `admin@ragflow.io/admin`）；
3. **HTTPS**：nginx 配置 `ragflow.https.conf`（443 TLS）；
4. **注册控制**：`REGISTER_ENABLED=0`、`DISABLE_PASSWORD_LOGIN=true`（改走 OAuth/OIDC）；
5. **RAGFLOW_SECRET_KEY**：多副本部署时显式设置（否则 Redis 集中生成共享）；
6. **沙箱**：确认仅绑 127.0.0.1、`SANDBOX_CONTAINER_NETWORK=none`、设 `SANDBOX_EXECUTOR_MANAGER_API_TOKEN`；
7. **传输加密**：`RAGFLOW_CRYPTO_ENABLED=true`（aes-256-cbc/sm4-cbc）可加密对象存储内容。

## 3. 性能调优参数

| 参数 | 默认 | 建议 |
|---|---|---|
| `MEM_LIMIT` | 8GB | ES/Infinity 容器内存上限，数据量大时上调 |
| `DOC_BULK_SIZE` | 4 | 写入批次，吞吐不足时加大 |
| `EMBEDDING_BATCH_SIZE` | 16 | 向量化批次 |
| WS（worker 数） | 1 | `launch_backend_service.sh` 拉起的 executor 进程数，按 CPU 调 |
| `MAX_TASKS_PER_WORKER` | — | worker 自愈回收阈值，GPU 碎片时调小 |
| MySQL `max_connections` | 1000 | 多 worker 时确认余量 |
| 吞吐极致（opt-in） | — | `MYSQL_FLUSH_LOG_AT_TRX_COMMIT=2` + `MYSQL_SYNC_BINLOG=0`（牺牲崩溃安全，注释实测提交 68ms→1.6ms） |
| 长文档解析慢 | — | 拆分上传；或 Layout recognition 换外部引擎（MinerU/Docling） |

## 4. 监控与可观测

- 健康检查：`GET /api/v1/system/healthz`（db/redis/doc engine/storage 四项）；
- Admin 服务注册表：`http://host:9381` 查看各服务健康/资源/uptime，可远程重启；
- 日志：容器 `docker logs`；`.env` 调 `LOG_LEVELS`；OTel 追踪 profile 加 `,jaeger`（UI 16686）；
- 统计：`GET /api/v1/system/stats`。

## 5. 故障排查速查表

| 症状 | 首查 | 处置 |
|---|---|---|
| 启动即退，日志见 ES 连接失败 | `vm.max_map_count` / MEM_LIMIT | sysctl 调参；上调内存 |
| 解析一直 RUNNING 不动 | executor 是否存活（`docker ps`） | 看 executor 日志；确认 embedding 可达 |
| 解析 FAIL：embedding 报错 | 模型 Key/额度/网络 | Model Providers 里"连接测试" |
| 检索测试有分但对话说"没找到" | Chat 相似度阈值过高 / Top N 太小 | 调低阈值、调大 Top N、开 rerank |
| 对话引用点开无截图 | 该块来自非 DeepDoc 解析 | 用 DeepDOC 或外部视觉引擎重新解析 |
| 上传 >1GB 失败 | `MAX_CONTENT_LENGTH` 与 nginx `client_max_body_size` | 两处同步调大 |
| Redis 起不来 / 队列堆积 | 6379 端口、valkey 容器 | `docker compose logs redis`；重启 |
| Go/Python 端点行为不一致 | 确认 `API_PROXY_SCHEME` 与实际端口 | 用 `/api/v1/language` 探测当前后端 |
| 数据库迁移报 1091（索引不存在） | 两侧索引名差异 | 属预期现象（列集合互认），勿手工改索引名 |
| 忘记 admin 密码 | `logs/admin_bootstrap_password.txt`（随机生成时 0600） | 或重设 `ADMIN_DEFAULT_PASSWORD` 后重启 |

## 6. 数据备份

```bash
# 元数据（必须）
docker exec docker-mysql-1 mysqldump -uroot -p'infini_rag_flow' rag_flow > ragflow_meta.sql
# 对象存储（原始文件/图片）：备份 minio 数据卷
docker run --rm -v docker_minio-data:/data -v $(pwd):/backup alpine \
  tar czf /backup/minio-data.tgz /data
# 检索引擎可重建（重新解析），一般无需备份
```

恢复顺序：起基础设施 → 导入 MySQL → 恢复 MinIO 卷 → 起主服务 →（如索引丢失）批量重新解析。

> 深读：compose 服务矩阵与全部 `.env` 项见 [project-analysis/08-deployment-and-ecosystem.md。
