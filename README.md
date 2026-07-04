# 镜像仓库文件（部署到 `dl.tkdl.cc` 背后的 GitHub 镜像仓库）

本目录是 **qopad 自建镜像 `dl.tkdl.cc` 的 GitHub Actions 镜像仓库**的文件蓝本（版本化托管在 qopad 仓库里作单一真相源）。
把本目录内容**原样拷进你的镜像仓库根**（即托管 `mirror.yml` 那个 public 仓库），即可让镜像同时覆盖：

- **Claude Desktop**：`win-x64` + `win-arm64`（MSIX）+ `mac`（universal dmg）
- **claude-code 二进制**（Code 模式的 `claude.exe.zst`，从 MSIX 解析 CD 钉死版本）
- **Codex App**：`win-x64` + `win-arm64`（MSIX，经微软商店 FE3 解析）+ `mac-arm64` + `mac-x64`（dmg）

## 目录 → 镜像仓库对应

```
docs/mirror-repo/                          你的镜像仓库根/
├── .gitattributes                  ───►  .gitattributes                （强制 LF；务必一起拷，防 CRLF 污染 CI bash 脚本）
├── .github/workflows/mirror.yml    ───►  .github/workflows/mirror.yml   （合并工作流：claude + codex 两个 job）
└── scripts/store-link/                    scripts/store-link/           （Codex Windows FE3 解析器，自包含 .NET，零 NuGet 依赖）
    ├── Program.cs                  ───►      Program.cs
    └── StoreLink.csproj            ───►      StoreLink.csproj
```

> ⚠️ **务必连同 `.gitattributes` 一起拷贝**：workflow 的 bash `run:` 步骤在 ubuntu runner 执行，若文件被 Windows 编辑器写成 CRLF，`\r` 会让 shell 报错。`.gitattributes` 强制 `*.yml`/脚本恒 LF。

> 若你镜像仓库里已有旧的 `mirror.yml`（只镜像 Claude），**用本 `mirror.yml` 整体替换**——它已包含原 Claude Desktop + claude-code 全部逻辑，并新增了 Claude arm64 与整个 Codex job。

## 前置（多数已就绪）

1. **R2 Secrets**（镜像仓库 → Settings → Secrets and variables → Actions）：`R2_ACCESS_KEY_ID` / `R2_SECRET_ACCESS_KEY` / `R2_ENDPOINT` / `R2_BUCKET`（与现有 Claude 镜像共用同一 bucket，无需新增）。
2. **.NET**：工作流用 `actions/setup-dotnet@v4` 自动装 8.0.x，无需手动准备。
3. **无需**任何微软账号/密钥：FE3 解析器直连微软商店公开端点（DisplayCatalog + FE3），只读、不鉴权。

## 部署 & 首跑

1. 拷贝上面两个路径到镜像仓库、`git commit && git push`。
2. 仓库 **Actions → mirror-desktop-apps → Run workflow**（手动触发首跑）。可在 `force_codex=true` 强制首次全量镜像 Codex。
3. 看日志：`claude` job 先跑（Claude MSIX x64/arm64 + mac + claude-code），`codex` job 后跑（FE3 解析 → 下载 Codex MSIX x64/arm64 + mac dmg → 传 R2）。

## 验证（镜像就绪后 curl）

```bash
# Claude Desktop arm64（新增）
curl -sI https://dl.tkdl.cc/claude-desktop/latest/win-arm64   # 200 + application/msix + Content-Disposition
curl -sI https://dl.tkdl.cc/claude-desktop/latest/win-x64     # 200（= win 别名同包）
# Codex App（新增，qopad codex-desktop 主源）
curl -sI https://dl.tkdl.cc/codex-desktop/latest/win-x64      # 200 + application/msix（~640MB）
curl -sI https://dl.tkdl.cc/codex-desktop/latest/win-arm64    # 200 + application/msix
curl -sI https://dl.tkdl.cc/codex-desktop/latest/mac-arm64    # 200 + application/x-apple-diskimage
curl -sI https://dl.tkdl.cc/codex-desktop/latest/mac-x64      # 200
curl -s  https://dl.tkdl.cc/codex-desktop/latest/manifest     # {win_ver, artifacts{sha256...}, ready[...]}
curl -s  https://dl.tkdl.cc/codex-desktop/latest.json         # {version, codex_version, mirrored_at}
```

## 首跑常见失败（排错）

| 现象（日志） | 原因 & 解法 |
|---|---|
| `codex` job 卡「Resolve + download Windows MSIX (FE3)」→ `预构建 store-link…` 后 exit 1 | **十有八九是只拷了 `mirror.yml`、没拷 `scripts/store-link/`**。工作流已加前置检查会明确报「缺少 store-link」——把 qopad 仓库 `docs/mirror-repo/scripts/store-link/`（`Program.cs` + `StoreLink.csproj`）整个目录拷进镜像仓库根再重跑。 |
| `::error::store-link 编译失败` + 上方 dotnet 报错 | `.NET SDK` 版本不符（csproj 目标 `net8.0`，工作流 `setup-dotnet` 装 `8.0.x`）或 `Program.cs` 被编辑器改坏（CRLF/编码）。看 dotnet 具体报错；确认 `.gitattributes` 也拷了（强制 LF）。 |
| `::error::Codex win-x64 FE3 失败`（逐次 store-link stderr） | 微软商店限流/网络/未上架该架构。工作流会逐次打印 store-link 完整 stderr 定位；**mac 仍会照常镜像**，下轮 12h 自动重试 win。 |
| `codex` job 红但 mac 短链已更新 | 正常——win FE3 失败标红以便留意，但 mac 下载 + 上传就绪产物走 `if: always()` 不受影响（见「逐产物就绪判定」）。 |

> 排错要点：FE3 步骤**不再隐藏 dotnet 输出**，编译/解析失败都会在日志打印具体报错；`Search logs` 搜 `::error` 直达。

## 设计要点（健壮性 / 版本策略）

- **增量镜像**：每 12h 探测上游版本（Claude=307 redirect 里的版本号；Codex=`windows-store-update.json` 的 `buildVersion` + mac dmg ETag 组成 signature）；**版本未变整轮 no-op**，变了才拉新版传 R2。
- **版本保留可回滚**：`archive/<ver>/` 保留最近 N 版（Claude 安装包 12 版；claude-code 16 版；Codex 因单版 ~2.3GB 只留 3 版）；`latest/*` 恒指最新供 qopad 消费。
- **版本一致性护栏（Codex win）**：FE3 解析出的 MSIX `moniker` 必须匹配探测到的 `buildVersion`+架构，否则判为「store 版本漂移」重试——保证镜像的正是当前商店版（与 qopad 经 winget 探测到的「可用版本」对齐，避免升级循环）。
- **逐产物就绪判定**：某架构/平台本轮下载失败**只跳过它、保留 R2 旧包**，不拖垮其它产物（x64 必成；arm64 / mac 单架构失败仅 `::warning::`）。
- **上传顺序**：先归档 → checksums/manifest → **latest 短链最后传**，避免 latest 指向半更新的包。
- **全链路重试**：探测/下载用 curl_cffi 轮换 TLS 指纹（绕 Cloudflare/上游 WAF 间歇 403）；FE3 解析+下载一体重试（签名 URL 会过期，每次重试重解析拿新链）；R2 `aws s3 cp` 自带分片+重试；`git push` 非快进自动 fetch+rebase 重试（**绝不 --force**）。
- **两 job 串行零竞争**：`codex needs claude` 排在其后、各推各的 manifest（`manifest.json` / `codex-manifest.json`）；`concurrency.group` 让多次 run 排队，绝不并发提交同仓库。`codex if: always()` 解耦：claude 失败也照镜像 codex。

## 与 qopad 的对接（已在 qopad 侧改好）

qopad `manifest.rs` 的 `claude-desktop` / `codex-desktop` 已改为按机器架构 `{arch}` 取 `latest/win-{arch}`、`latest/mac-{arch}`，并把 `dl.tkdl.cc` 设为**主源**（Codex 用 `prefer_first` 钉在 winget 之前），第三方 `agentsmirror` 作 backup。

- **上线顺序无强依赖**：本工作流未部署/未跑完时，qopad 请求 tkdl 短链 404 会**快速降级到 agentsmirror**（仍可装），不影响用户。工作流跑通后 tkdl 就位、自动成为主源。
- **Codex 版本更新机制**：qopad 经 `winget upgrade --include-unknown --source msstore` **只读探测**商店最新版；升级则重下 tkdl `latest/win-{arch}` MSIX。故镜像 `latest/*` 越贴近商店最新越好——12h cron 是「更新频率」与「新鲜度」的折中，最长约 12h 追平（期间 qopad 可能短暂显示「有新版」，属预期）。

## 合规

逐字节原样镜像官方免费安装包，不改/不重打包/不破解；`manifest`/`checksums` 公布 SHA256 可比对官方；仅加速下载、不碰鉴权；留下架通道（详见主蓝图 `docs/CLAUDE_DESKTOP_MIRROR_SELFHOST.md` §5）。
