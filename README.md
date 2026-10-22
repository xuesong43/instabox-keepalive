# instabox-keepalive

用 GitHub Actions 每 5 分钟对 instabox 沙箱做一次 SSH 握手，避免实例因闲置被暂停/回收。

## 原理

instabox 的实例在无活动时会被 `paused` / `stopped`。这个仓库的定时任务只做一件事：

```
ssh -p 2222 <token>@ssh.instabox.sh 'echo alive; uptime'
```

握手成功即代表实例被"碰"了一次。任务本身不修改实例内任何内容。

## 配置（只做一次）

1. 打开仓库 **Settings → Secrets and variables → Actions → New repository secret**
2. Name 填 `INSTABOX_SSH_USER`
3. Secret 填你的 SSH 用户名（形如 `ibssh_xxxxxxxx`，**这串就是凭证本身**，不是普通用户名）
4. 到 **Actions → instabox-keepalive → Run workflow** 手动跑一次，日志里出现 `alive at ...` 就说明通了

## 换实例 / 轮换凭证

只改 Secret 即可，不需要动代码：

- 换实例：把 `IB_HOST` / `IB_PORT` 改成新实例的值（在 `keepalive.yml` 的 `env` 里）
- 轮换 token：直接更新 Secret `INSTABOX_SSH_USER`

## 为什么用公开仓库

GitHub Actions 对**公开仓库**完全免费；私有仓库免费额度只有 2000 分钟/月，而每 5 分钟一跳按月计约 `12 × 24 × 30 ≈ 8640` 分钟（每个 job 至少计 1 分钟），会直接超额。仓库公开不影响安全：token 存在 Secret 里，Secret 不出现在代码、日志、fork 中，其他人只看到本文件和一个 SSH 命令。

## 已知限制

- **60 天不活动会被自动禁用**：公开仓库的 `schedule` 在仓库连续 60 天无提交时会停用，需回 Actions 页面点一次 Enable，或随便提一次 commit。
- **触发有延迟**：GitHub 的 cron 在高峰期可能延迟 5-15 分钟，且不允许低于 5 分钟。如果实例的闲置阈值比这个还短，需要换成本机计划任务或 cron-job.org。
- **救不了额度停机**：`spend cap` 达到时实例会被服务端硬停（`machines are stopped until next month or a higher cap`），心跳无法阻止。
- **不含交互式会话**：本任务只做短连接握手。如果平台按"活跃 SSH 会话"判定活跃，短连接可能不算，需要改成常驻连接方案。
