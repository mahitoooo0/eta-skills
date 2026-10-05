---
name: requesting-code-review
description: Use when completing tasks, implementing major features, or before merging to verify work meets requirements（完成任务、实现重大功能或合并前，需要派 review 子代理核验工作时使用）
---

# 请求代码评审（Requesting Code Review）

派一个 review 子代理，在问题级联扩散之前把它抓出来。评审者拿到的是精心裁剪过的上下文，绝不是你的会话历史。

**核心原则：** 早评审，常评审。

## 何时请求评审

**必须：**
- 子代理驱动开发流程中，每完成一个任务之后
- 完成重大功能之后
- 合并之前（`manage_agent_workspace` 做 merge 之前，必须已经有 review 子代理在同一个 workspace_id 上跑完并被核验）

**可选但值得：**
- 卡住的时候（需要一双新眼睛）
- 重构之前（建立基线）
- 修完复杂 bug 之后

## 怎么请求（Eta 流程）

### 1. 主代理先把评审材料准备好

子代理不能执行 shell，评审者不会自己跑 `git diff`、`git log`、`git status`。凡是需要命令的地方，都由**主代理**用 `terminal`（environment=android/linux）在仓库根执行，然后把结果交给子代理。

主代理执行并留存的材料：

```bash
git rev-parse HEAD~1            # 或 git merge-base origin/main HEAD，作为 BASE
git rev-parse HEAD              # 作为 HEAD
git diff --stat BASE..HEAD
git diff BASE..HEAD
git log --oneline BASE..HEAD    # 需要时补
```

如果被审的变更还在 implementation 子代理的隔离 worktree 里（平台自动创建，主代理用 `manage_agent_workspace` 管理），则把 **workspace_id** 和仓库根路径一起写进评审上下文，由主代理决定把哪些 diff 内容直接粘进 context。

### 2. 派 review 子代理

用 `delegate_task(worker=1..6, role="review", project="/workspace/<git 仓库根>", context=...)`，context 按本目录 [code-reviewer.md](code-reviewer.md) 模板填好。

硬性要求：
- 必须带 `project`（`/workspace/<git 仓库根>`）
- 必须带要审的 `workspace_id`，否则评审对象不明
- context 里放：实现摘要、计划/需求、diff 内容（或其在 worktree 中的路径）、相关接口约定
- 评审者只能 `read_file` / `list_directory` 只读检查产物；它不能执行 shell、不能开浏览器、不能读写它自己 worktree 之外的东西，**也不能再派子代理**
- 评审者需要额外材料（某个文件全文、另一段 diff、依赖清单）时，主代理用 `terminal` 取好再送过去；任务还在跑就用 `supervise_task(guide)` 给它补指令
- 用 `get_task_result` 取评审结论；任务失败且已停止、`can_replace=true` 时才能换一个评审者，同一份工作不要重复派发

### 3. 处理反馈

- Critical 问题立即修
- Important 问题修完再继续
- Minor 问题记进待办清单
- 评审者说错了，要有技术理由地反驳（见兄弟技能 `receiving-code-review`）

## 占位符

- `{DESCRIPTION}` — 你做了什么，简要总结
- `{PLAN_OR_REQUIREMENTS}` — 它应该做什么（计划要点、任务原文或需求）
- `{WORKSPACE_ID}` — 被审变更所在的隔离工作区
- `{REPO_ROOT}` — 仓库根，例如 `/workspace/eta-skills`
- `{DIFF_OUTPUT}` — 主代理用 terminal 生成的 diff 文本；内容太长时给 worktree 内的文件路径
- `{BASE}` / `{HEAD}` — 变更起止提交

## 示例

```
[主代理刚完成 Task 2：新增校验函数]

主代理: 先评审再继续。

主代理用 terminal 执行:
  git rev-parse HEAD~1  → a7981ec
  git rev-parse HEAD    → 3df7661
  git diff a7981ec..3df7661 > /tmp/review-task2.diff

主代理 delegate_task(worker=1..6, role="review", project="/workspace/eta-skills", context=...)
  DESCRIPTION: 新增 verifyIndex() 与 repairIndex()，覆盖 4 种问题类型
  PLAN_OR_REQUIREMENTS: 计划文档里的 Task 2
  WORKSPACE_ID: 该 implementation 任务的隔离工作区标识
  DIFF_OUTPUT: 主代理执行 git diff 的输出

[评审子代理返回]:
  Strengths: 架构清晰，测试是真实测试
  Issues:
    Important: 缺少进度提示
    Minor: 报告间隔用了魔法数字 100
  Assessment: 可以继续

主代理: [先修进度提示]
[进入 Task 3]
```

## 常见自我辩解（Common Rationalizations）

| 借口 | 事实 |
|--------|---------|
| “我自己把 diff 读一遍就行，不用派评审者” | 你是操盘手，内联评审大 diff 会烧掉你继续推进工作所需的上下文。派 `role="review"` 子代理：diff 和判断都留在它的上下文里，只有结论回到你这里。 |
| “评审者需要我的整段会话历史才能理解这次改动” | 给它精心裁剪的上下文，绝不给会话历史。这样评审者盯着工作产物，而不是你的思考过程。 |

## 危险信号（Red Flags）

**绝不要：**
- 因为“很简单”就跳过评审
- 无视 Critical 问题
- 带着未修的 Important 问题继续推进
- 对成立的技术反馈争辩

**如果评审者错了：**
- 用技术理由反驳
- 拿出代码/测试证明它确实能工作
- 请它说明依据

模板见本目录：[code-reviewer.md](code-reviewer.md)
