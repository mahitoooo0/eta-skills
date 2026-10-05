---
name: coding-repo-workflow
description: 手机上完成编程任务的主线方法：获取代码、修改、验证、提交回写、CI 监控。当用户要求修改仓库、修 bug、加功能、提 PR、分析项目代码时使用。
---

# Coding Repo Workflow

在手机上完成编程任务的完整闭环：取码 → 改 → 验证 → 回写 → CI 监控。每一步都有多种路径，按环境实际可达性选择，不要在一条路上反复重试。

## 第一步：定位与获取代码

按顺序尝试，成功即止：

1. **共享目录已有项目**：先 `ls /workspace/mounts/` 确认用户共享的目录里有没有目标项目；有就直接在 Linux 环境里工作。
2. **git clone**：`git clone --depth 1 <url>`（走代理网络通常可达；超大仓库加 `--filter=blob:none`）。clone 卡住超过 2 分钟就换下一条路径，不要干等。
3. **GitHub API 逐文件**：`api.github.com` 直连最可靠。先 `GET /repos/{o}/{r}/git/trees/{branch}?recursive=1` 拿文件树，再 `GET /repos/{o}/{r}/contents/{path}` 读单个文件（返回 base64，需解码）。具体命令模板见 `references/github-api.md`。

## 第二步：修改纪律

1. 改共享文件前，**先从远端拉当前最新版本**，在最新版上施加改动；本地旧副本不可信。
2. 小步修改：一次只解决一件事，改完即验，不把多个改动混在一起。
3. 修改会波及调用点时（改函数签名、加枚举值、删字段），先全仓搜索引用；语言有穷举检查（如 Kotlin `when`）时靠编译兜底，不靠人眼找。
4. 批量替换前先备份原文件，改完 diff 逐行确认只动了预期位置。

## 第三步：验证（分级选择）

| 层级 | 适用 | 做法 |
|------|------|------|
| 本地快速验证 | 脚本、纯逻辑、单文件 | 在 Linux 环境直接跑：语法检查、单测、冒烟脚本 |
| 本地构建 | 依赖齐全的小项目 | 在 Linux 环境装工具链后构建；装包失败先看 linux-workspace-ops 技能 |
| 远端 CI | Android/大型项目、本地装不动 SDK | push 到非 main 分支或派发 `workflow_dispatch`，用 CI 兜底 |

远端 CI 的关键规则：

- `workflow_dispatch` 要求该 workflow 文件**在目标 ref 上存在**，否则 404/422；先查清楚再派发。
- 派发后用 `gh run watch <id> --exit-status` 阻塞等待；失败立刻 `gh run view <id> --log-failed` 定位，不要重跑碰运气。
- Actions 缓存按分支隔离：依赖分支缓存的构建（如签名 keystore）在别的分支跑会拿到不同结果，出包类构建固定在专用分支上跑。

## 第四步：提交与回写

- **优先 gh CLI**：没装就先装（npm 或 apt 视环境）；有 token 时 `curl -H "Authorization: Bearer $GITHUB_TOKEN" api.github.com/...` 等价可用。
- **单文件改动**：`PUT /repos/{o}/{r}/contents/{path}`，body 带 `message`、`content`（base64）、`sha`（旧文件sha）、`branch`。
- **多文件改动**：用 Git Data API 打**单个提交**（blobs → tree(base_tree) → commit(parents=head) → PATCH ref），避免多次 push 触发多次构建互相 cancel。步骤化命令见 `references/github-api.md`。
- 大内容（几十 KB 以上）不要放进命令行参数（会 "Argument list too long"）：用脚本构造 JSON 写入临时文件，再 `gh api --input <file>` 或 `curl -d @file`。
- **绝不 force push；绝不直接 push main 分支**——push main 前必须先向用户确认目标分支和改动范围。

## 第五步：与子代理配合

- 多文件调查、跨模块分析 → 同一轮并行派 `research` 子代理按模块分读，主代理汇总。
- 改动完成后 → 派 `review` 子代理（给它 project 和 workspace_id）独立审查 diff。
- 编排细节见 subagent-orchestration 技能；子代理不可用时主代理自己做，并在回答中说明。

## 红线

- 不把 token、密钥写进代码、回复或仓库文件；发现泄露立即提醒用户撤销。
- 不绕过 CI 失败强推；不删别人的 workflow/secrets。
- 未经用户确认不创建 release、不关闭 issue/PR、不删仓库内容。
