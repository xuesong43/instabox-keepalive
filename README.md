# instabox-keepalive

用 GitHub Actions 每 5 分钟对 instabox 沙箱做一次 SSH 握手，避免实例因闲置被暂停/回收。

## 原理

这个 workflow 只做两件事：

1. **每 5 分钟**握手一次：`ssh -p 2222 <token>@ssh.instabox.sh 'echo alive; uptime'`
2. **每天首次运行**时提交一次 `.state/last-active`，日期没变就跳过

第 2 件事是为了绕开 GitHub 的一条规则：

> In a public repository, scheduled workflows are automatically disabled when no repository activity has occurred in 60 days.
> —— GitHub 官方文档（`events-that-trigger-workflows`）

## 为什么不是每次运行都提交

5 分钟一次 × 24 × 60 ≈ **288 个 commit/天，约 8.6 万个/年**。这会把提交历史、Actions 列表、仓库体积全部污染，而且高频 push 也容易触发限流。

60 天规则只要求"60 天内至少有一次活动"，**每天一次就是 60 倍余量**，再密没有任何收益。

另外两点设计：

- 提交步骤用 `if: always()`，所以即使某次 SSH 握手失败，当天的活动标记仍会打上——沙箱临时故障不该把定时任务连坐停掉。
- 用 `GITHUB_TOKEN` 推送的提交**不会**再触发本 workflow（避免自激循环），这也正是 `push` 触发器不会无限套娃的原因。

## 配置（只做一次）

1. 打开仓库 **Settings → Secrets and variables → Actions → New repository secret**
2. Name 填 `INSTABOX_SSH_USER`
3. Secret 填你的 SSH 用户名（形如 `ibssh_xxxxxxxx`，**这串就是凭证本身**，不是普通用户名）
4. 到 **Actions → instabox-keepalive → Run workflow** 手动跑一次，日志里出现 `alive at ...` 就说明通了

仓库的 **Settings → Actions → General → Workflow permissions** 需要是 **Read and write**，否则每日活动标记推不上去。

## 换实例 / 轮换凭证

只改 Secret 即可，不需要动代码：

- 换实例：把 `IB_HOST` / `IB_PORT` 改成新实例的值（在 `keepalive.yml` 的 `env` 里）
- 轮换 token：直接更新 Secret `INSTABOX_SSH_USER`

## 为什么用公开仓库

GitHub Actions 对**公开仓库**完全免费；私有仓库免费额度只有 2000 分钟/月，而每 5 分钟一跳按月计约 `12 × 24 × 30 ≈ 8640` 分钟（每个 job 至少计 1 分钟），会直接超额。仓库公开不影响安全：token 存在 Secret 里，Secret 不出现在代码、日志、fork 中，其他人只看到本文件和一个 SSH 命令。

## 已知限制

- **触发有延迟**：GitHub 的 cron 在高峰期可能延迟 5-15 分钟，且不允许低于 5 分钟。如果实例的闲置阈值比这个还短，需要换成本机计划任务或 cron-job.org。
- **救不了额度停机**：`spend cap` 达到时实例会被服务端硬停（`machines are stopped until next month or a higher cap`），心跳无法阻止。
- **不含交互式会话**：本任务只做短连接握手。如果平台按"活跃 SSH 会话"判定活跃，短连接可能不算，需要改成常驻连接方案。
- **活动标记按 UTC 换日**：北京时间每天 08:00 打点一次。
