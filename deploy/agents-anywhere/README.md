# 跑起来：Agents-Anywhere 服务端 + Postgres + Redis

镜像由[本仓库 CI](../../.github/workflows/agents-anywhere.yml)构建：
`ghcr.io/<owner>/agents-anywhere`。**多架构**（amd64 + arm64），`docker compose` 会按本机架构自动选层，
arm 服务器不需要开 QEMU 模拟。这里只管**服务端**；连接器、桌面/手机客户端是另一回事
（客户端去[上游下载页](https://www.agents-anywhere.com/download)，连接器装在被管理的机器上）。

| 组件 | 镜像 | 作用 | 端口 | 持久化 |
|---|---|---|---|---|
| `server` | `ghcr.io/<owner>/agents-anywhere` | FastAPI 后端 + 静态导出的 Web 控制台（同源） | `8000`（compose 映射到 5174） | `/data`：上传/附件 |
| `postgres` | `postgres:17-alpine` | 唯一的事实来源（会话、Timeline 等） | `5432` | `/var/lib/postgresql/data` |
| `redis` | `redis:8-alpine` | 跨实例协调、Pub/Sub、Timeline 序号与写缓冲 | `6379` | `/data`（AOF，**必须持久化**） |

Redis 不是可选的：上游文档写明那些 key **没有 TTL**，被驱逐就等于丢数据 —— 所以 AOF +
持久卷 + `noeviction`，这三样在 compose 里都写死了。

## 跑法

```bash
cp .env.example .env
# 编辑 .env：AGENTS_ANYWHERE_IMAGE 换成你的镜像地址，两个密码用 openssl rand -hex 32 生成
docker compose up -d
docker compose logs -f server      # 首次空库会打印引导 token
```

然后开 `http://127.0.0.1:5174`，用日志里的 token 创建第一个管理员。

启动顺序：`postgres`/`redis` 健康 → `migrate` 跑一次迁移并退出 → `server` 起来。
`migrate` 和 `server` 用的是同一个镜像，只是换了个 command。

## 三个容易踩的点（上游文档明确写的）

1. **首次必须在空库上用单 worker 完成引导**：初始引导 token 只存在于那个进程里，日志里打印。
   想开多 worker（`AGENT_SERVER_WORKERS`，需同时 `AGENT_SERVER_TIMELINE_SINGLE_INSTANCE=false`）
   或加副本，先完成引导。
2. **升级不能新老写者并存**：同一条库上不要让新旧版本的写者同时跑 —— 要「停旧 → 备份 → 迁移 → 只起新」。
3. **大版本迁移可能锁表**：例如序号列从 `int4` 扩到 `int8` 会 `ALTER TABLE`，可能重写表或索引，
   生产要留维护窗口、先在等规模副本上演练。

## 想接外部 PG / Redis

把 compose 里 `postgres` / `redis` 两个服务删掉，同时删掉 `migrate` / `server` 上指向它们的
`depends_on`（留着会报 `depends on undefined service`），再把 `AGENT_SERVER_DB_URL` /
`AGENT_SERVER_REDIS_URL` 换成外部地址。外部 Redis 同样得是 AOF + `noeviction` + 持久盘。

更细的参数（`AGENT_SERVER_DB_POOL_SIZE`、S3 文件存储、OAuth 回调等）见上游
[`server/README.md`](https://github.com/anywhere-labs/Agents-Anywhere/blob/main/server/README.md)。
