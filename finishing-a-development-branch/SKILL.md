---
name: finishing-a-development-branch
description: Use when implementation is complete, all tests pass, and you need to decide how to integrate the work. 实现完成、测试通过、需要决定如何合入工作时使用。
---

# 收尾一条开发分支

## 总览

**核心原则：** 核验测试 → 确认评审 → 呈现出口 → 执行选择 → 由平台清理 workspace。

**开始时声明：**"我正在用 finishing-a-development-branch 技能收尾这项工作。"

## 第 1 步：核验测试

由主代理用 `terminal(environment=linux)` 在目标 workspace 上跑项目的完整测试套件（`npm test` / `cargo test` / `pytest` / `go test ./...` 等），把运行结果作为证据。

**若测试失败**，报告失败并停下——菜单只在全套通过之后才出现：

```
Tests failing (<N> failures). Must fix before completing:

[展示失败]
```

**若测试通过：** 进入第 2 步。

## 第 2 步：确认 workspace 与评审状态

Eta 下实现由 implementation 子代理完成，平台会为它自动创建隔离的 git worktree。收尾前必须掌握：

- 本次工作的 `workspace_id`——主代理用 `manage_agent_workspace(list / inspect)` 查得。
- 本次工作从哪个分支分叉（即目标合入分支）。

**硬性前置：merge 之前，必须先有一个 review 子代理在这个 workspace_id 上跑完评审，且结论已核验。** 若还没跑，先派发 `delegate_task(role="review", project=/workspace/<git 仓库根>, context=...)`，拿到核验后的评审结论再继续。评审未完成时不得合入。

（评审子代理只能 `read_file` / `list_directory`，不能执行 shell、GUI、浏览器，也不能再派发子代理；需要 `git diff` / 测试结果时，由主代理用 terminal 取得后喂给它。）

## 第 3 步：确认目标分支

目标分支就是本次工作分叉出来的那个分支——通常写在计划、对话或分支上游里。若尚未确定，就问："这次工作是从 `<你的最佳猜测>` 分出来的——对吗？" 合入前确认：合错基线代价很高，难以撤销。

## 第 4 步：呈现出口

Eta 下**只有两条合法出口**，没有"保留分支、什么都不做"这个终点——那会让隔离 workspace 悬空泄漏。呈现恰好这两项：

```
实现完成。你希望怎么处理？

1. 合入 <目标分支>（我用 manage_agent_workspace merge 合入并清理 workspace）
2. 丢弃这次工作（我用 manage_agent_workspace discard 丢弃并清理 workspace）

选哪个？
```

按原文呈现这条菜单——简洁，每个选项都来自上面的列表。合入是默认建议。丢弃只在用户明确要求放弃工作、并逐字确认后执行（见第 5 步"出口 2"）。等用户的回答——集成决定权在用户。


## 第 5 步：执行选择

> **合入前检查清单（出口 1 必须先全部满足）：**
> - [ ] **测试通过**——在要合入的那棵树上跑过完整测试套件且全绿。
> - [ ] **评审完成**——review 子代理已在该 workspace_id 上跑完评审，结论已核验。
> - [ ] **遗留项已记录**——已知问题、TODO、未完成项都已写明，供用户知晓。

### 出口 1：合入

1. 确认上面三条前置全部满足。
2. 由主代理执行合入：`manage_agent_workspace merge`（fast-forward only），**必须传入上面那条 workspace_id**——若该 workspace 还没有跑过评审，先派 `delegate_task(role="review", project=<git 仓库根>, workspace_id=<该 workspace_id>)` 补跑并核验结论，再合入。平台负责把隔离 worktree 的改动并入目标分支并回收该 workspace。**不会自动 push。**
3. 合入后在目标分支上再跑一次测试套件做验证。若合入结果测试失败：停下，报告失败，保留现场供排查——由于没有 push，合入是本地且可恢复的。

### 出口 2：丢弃

这条路径只在用户明确要求把工作丢掉时、作为响应存在。先确认：

```
这会永久删除：
- 分支 <name>
- 全部提交：<commit-list>
- workspace <workspace_id>

输入 'discard' 确认。
```

等用户逐字输入确认。收到后由主代理执行 `manage_agent_workspace discard` 丢弃并回收该 workspace。

## 第 6 步：清理

**清理由平台负责。** `manage_agent_workspace merge` 与 `manage_agent_workspace discard` 都会在完成后回收该 workspace 的隔离 worktree。主代理无需、也不应手动执行任何 `git worktree` 相关命令（如 `git worktree remove` / `git worktree prune`）；这些命令在本技能里已全部删除。

## 快速参考

| 出口 | 合入 | 清理 workspace |
|--------|-------|----------------|
| 1. 合入 | 是（fast-forward，不自动 push） | 是（由 `manage_agent_workspace merge`） |
| 2. 丢弃（仅在用户明确要求时） | - | 是（由 `manage_agent_workspace discard`） |

## 常见合理化

| 借口 | 现实 |
|--------|---------|
| "本会话早些时候测试通过了" | 在你要合入的那棵树上跑测试套件。一次绿灯只证明它跑过的那棵树。 |
| "他们显然想合入" | 集成是用户的决定。呈现出口并等待。 |
| "看起来这功能已经做完了——我提议丢弃它" | 出口就这写好的两条。丢弃只在用户逐字要求时发生。 |
| "'行，删了吧'也算确认" | 只有逐字输入的 `discard` 才授权删除。 |
| "先保留分支、以后再处理行不行" | 没有这个终点。悬空的隔离 workspace 会泄漏。要么合入，要么丢弃，二选一。 |
| "合入结果失败大概是偶发" | 合入结果失败就停一切。保留现场排查；合入是本地且可恢复的。 |
| "目标分支显然是 main" | 确认分叉点或直接问。合错基线代价很高、难以撤销。 |
| "评审还没跑，但代码看着没问题，先合了" | merge 前必须先在同一个 workspace_id 上跑完 review 并核验。没跑完就不合。 |
| "我自己去 `git worktree remove` 把它清掉就行" | 不。清理归 `manage_agent_workspace`。手动动 worktree 会破坏平台状态。 |
