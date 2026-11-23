# 上游镜像自动打包

把**没有官方镜像**的上游项目：定时检查更新 → 自动构建 → 推送镜像。

| 镜像 | 上游 | 流水线 | 平台 |
|---|---|---|---|
| `ghcr.io/<owner>/agents-anywhere` | [`anywhere-labs/Agents-Anywhere`](https://github.com/anywhere-labs/Agents-Anywhere) 的 GitHub Release | [`agents-anywhere.yml`](.github/workflows/agents-anywhere.yml) | amd64 + arm64 |

> **本仓库不含任何上游源码。** 源码只在构建时按 release tag 现取，用完即弃。
> `deploy/` 下是「怎么把镜像跑起来」的说明。

---

## 镜像在哪

| 项 | 值 |
|---|---|
| 镜像 | `ghcr.io/<owner>/agents-anywhere` |
| 推送凭据 | 内置 `GITHUB_TOKEN`，不需要额外配 Secret |
| 可见性 | 公开仓库推出来的包默认公开；`pull` 报 401 就去 Packages → 改可见性（只能在网页上改） |

标签：

| 标签 | 含义 |
|---|---|
| `latest` | 最新一份**正式** release 的构建 |
| `vX.Y.Z` | 对应上游 release tag |
| `sha-<前12位>` | 内容寻址，对应 release 指向的提交；也是判断「有没有更新」的依据 |
| `sha-<前12位>-amd64` / `-arm64` | 按架构的中间产物（合成多架构 manifest 用，调试也能单独拉） |

```bash
docker pull ghcr.io/<owner>/agents-anywhere:latest
```

平台 `linux/amd64` + `linux/arm64`，**两个架构各自原生构建**，不用 QEMU：

| 架构 | runner | 说明 |
|---|---|---|
| amd64 | `ubuntu-latest` | |
| arm64 | `ubuntu-24.04-arm` | GitHub 的 arm64 托管 runner，换自建机器设 `AGENTS_ANYWHERE_ARM64_RUNNER` |

上游构建里有 Next.js（node 阶段 `yarn build` + python 阶段 `uv sync`），在 QEMU 下模拟 arm64
会慢到不可用 —— 所以拆成两个 job 并行，各自用原生 CPU。两个 job 分别推
`<内容寻址>-amd64` / `<内容寻址>-arm64`，最后由 `merge` job 用 `docker buildx imagetools create`
合成多架构 manifest，`docker pull` 会按机器架构自动选层。

---

## 三种触发方式

| 方式 | 怎么触发 | 行为 |
|---|---|---|
| 自动 | 定时（`schedule`，每 6 小时） | 查上游最新正式 release；已构建过就跳过（只花十几秒） |
| 自动 | 推送本 workflow 文件（仅默认分支） | 同上。首次推上来会立刻出一份，不必等定时 |
| 手动 | Actions → **agents-anywhere** → **Run workflow** | 同上；勾 `force` 强制重建，填 `tag` 重建某个历史版本 |

```bash
gh workflow run agents-anywhere.yml -f force=true      # 强制重建最新 release
gh workflow run agents-anywhere.yml -f tag=v2.2.0      # 重建指定 release（不会动 latest）
```

> 「Run workflow」按钮要在本文件推到默认分支之后才出现 —— 定时任务同理。

---

## 它怎么工作

```
每 6 小时（cron）
   │
   ├─ 查 Release API：GET /repos/anywhere-labs/Agents-Anywhere/releases/latest
   │     → tag_name / published_at / html_url
   │
   ├─ 解析该 tag 指向的提交 → sha
   ├─ 问 GHCR：<image>:sha-<前12位>、-amd64、-arm64 三个都在吗？
   │     都在 → 内容没变，跳过（不构建）
   │     缺一个 → 往下走
   │
   ├─ [build ×2，并行]
   │   ├─ 下 codeload 的 tag 源码 tarball → 解到 src/（构建上下文必须是仓库根）
   │   ├─ 校验 Dockerfile 依赖的输入都在（web-next/package.json、.yarnrc.yml、server/pyproject.toml）
   │   ├─ 用**上游自带的 docker/Dockerfile**（target=server）构建 linux/<arch>
   │   └─ 推 <image>:sha-<前12位>-<arch>
   │
   └─ [merge] 两个中间 manifest 都在 → imagetools create 合成
        latest（仅当是最新 release）+ vX.Y.Z + sha-<前12位>
```

判断依据是**release 指向的提交**而不是 release tag 本身：维护者重新发布、把 tag 移动到别的
提交时也能认出来，不会误判成「已构建」。副作用是重复运行天然安全 —— 没变就是十几秒的事。

构建缓存走 GitHub Actions 的 `type=gha`（`scope=agents-anywhere`），yarn / uv 那几层没变就不重装。

---

## 首次使用

1. 把这个仓库推上 GitHub（公开仓库最省事）。
   推上来时本文件变化会触发一次构建，不用等定时。
2. `docker pull ghcr.io/<你的用户名>/agents-anywhere:latest`
3. 跑起来：见 [`deploy/agents-anywhere/`](deploy/agents-anywhere/README.md)（Postgres + Redis + 服务端）。

镜像名里的用户名自动取仓库所有者（含大写自动转小写，GHCR 不收大写名），所以 **fork 之后不用改任何文件**。

---

## 配置项

| 类型 | 名称 | 必需 | 说明 |
|---|---|---|---|
| Variable | `AGENTS_ANYWHERE_IMAGE_NAME` | 否 | 改镜像名，默认 `agents-anywhere` |
| Variable | `AGENTS_ANYWHERE_ARM64_RUNNER` | 否 | 换 arm64 构建用的 runner，默认 `ubuntu-24.04-arm` |

**改轮询频率**：编辑 `.github/workflows/agents-anywhere.yml` 里的 cron。

```yaml
  schedule:
    - cron: '17 */6 * * *'   # 每 6 小时；想每天一次就写 '17 3 * * *'
```

---

## 常见问题

- **上游没有新 release，但我想重建**：Run workflow 勾 `force`。
- **刚改成双架构后的第一次运行**：会重建一遍——旧的单架构流水线只推了 `sha-xxx`，
  没有 `-amd64` / `-arm64` 中间标签，所以判断逻辑会认为是「没建全」。
- **上游还很新、一个 release 都没有**：这条流水线会 `::error::` 退出（`/releases/latest` 404），
  不会产出半成品镜像。
- **构建失败在上游**：Dockerfile 或输入路径被上游改动时，工作流会在「下载源码」那步就明确报
  `::error::` 缺哪个文件，而不是闷头跑二十分钟再失败。
- **`latest` 没动**：只有「最新那份 release」的构建才会推 `latest`；重建历史版本不影响它。
- **只想重跑一个架构**：`build` job 用了 `fail-fast: false`，两个架构互不牵连；单边失败时
  下一次 `force` 重建会复用另一边的层缓存。
- **arm64 runner 不可用**（私有仓库且套餐不支持 arm64 托管 runner）：设 Variable
  `AGENTS_ANYWHERE_ARM64_RUNNER` 指向自建 arm64 runner，或把 workflow 里那一行删掉，
  退回单架构（arm 服务器就只能走 QEMU 模拟）。
- **GitHub 的 `schedule` 不保证准点**，也可能跳过某次 —— 平台行为，下一次照样会检出变化。

---

## 许可

本仓库的 workflow 与文档：MIT。

镜像里的 Agents-Anywhere 源码由上游以 **Apache-2.0** 授权，版权归原作者。这是**非官方镜像**，
镜像标签里记了 `org.opencontainers.image.licenses=Apache-2.0`、上游仓库、release tag 与提交，
可以逐字对回是哪个提交打出来的。
