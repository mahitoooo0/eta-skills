---
name: constraint-driven-development
description: Establishes a project's quality bar as a written contract and stops agents quietly lowering it. Interviews the user on which dimensions matter, supplies sane default thresholds when they have no number in mind, records everything in CONSTRAINTS.md, and watches the diff for a weakened bar — new @ts-ignore or eslint-disable suppressions, skipped or deleted tests, assertions stripped out, unimplemented stubs, thresholds edited down. Use when no quality bar is written down, when the user says "set up constraints" or "define our standards", when the user wants dimensions they care about — accessibility, web performance, coverage — set up as enforced constraints, when an agent keeps silencing checks or skipping tests to get to green, when you need a coverage or performance threshold and don't know what number to pick, or when an agent writes more code than anyone will read. 在 Eta 里由主代理在每次 implementation 子代理交回后用 terminal 执行检查。
---

# 约束驱动开发（Constraint-Driven Development）

## 概述

这个技能包里别的技能描述"好"长什么样。`code-review-and-quality` 给你五个轴。`test-driven-development` 给你一个循环。`security-and-hardening` 给你一份威胁清单。所有这些都活在 agent 会读、但可能读也可能不读的散文里，而且没有任何一条能撑过会话结束。

本技能产出的东西不同：一份关于**本项目**标准线的书面记录，带数字，它能比对话活得久，并且可以被机械地检查。

原因很重要。当你自己写代码时，读一遍就知道它好不好。一个 agent 一个下午写出的量超过你那一周会读的量，于是判断从你脑子里搬到绕着循环运行的检查里。那些检查必须存在，必须有你真正选定的数字，而且必须离工作足够近，近到 agent 能修好自己的产出。

规格驱动开发说明要造什么。测试驱动开发证明它能工作。约束驱动开发在任何人于 PR 里争论之前，就先定义了"足够好到可以发布"意味着什么。

## 何时使用

在以下情况应用本技能：

- 启动一个项目或一个重大特性，而没有任何成文的标准线
- 用户要求"设置约束""加质量门""定义我们的标准"，或"别再让 agent 发垃圾"
- 一个 agent 产出的量没人逐行在读
- CI 里有检查，但没人能说清哪些拦合并、哪些只是装饰
- 覆盖率、性能或无障碍的数字在每个 PR 里被争论，而不是一次定好
- 你即将进行一轮自主实现循环（由主代理把活派给 implementation 子代理），而横在它和主干之间的唯一东西是 agent 自己也写了的一套测试

**何时不要用：**

- 项目已有 `CONSTRAINTS.md`，且用户没在改它——照读照做即可
- 一次性脚本、探针、用完即弃的原型
- 用户现在就要一份代码审查（`code-review-and-quality`），或要搭建流水线（`ci-cd-and-automation`）
- 产品市场契合之前的代码、预期存活只有两周——下面的 floor 仍然值得，其余不值得

## 加载约束（Loading Constraints）

访谈需要一个在线的用户。**不要在非交互场景里跑访谈**（纯批量委派、无人应答的自主运行）。如果约束缺失而你正处在这样的场景里，应用下面的 Floor，注明你这么做了，并把其余部分标记给人工处理。

需要读文件或跑命令来采集事实时，由主代理用 `read_file` / `list_directory` / `terminal` 执行；需要跑命令的地方不接受由子代理执行。

## 流程

### Step 1: 先探测，再提问

绝不问你本可以读到的信息。在问第一个问题之前，先收集：

| 什么 | 去哪里看 |
|------|-------|
| 语言与栈 | `package.json`、`pyproject.toml`、`go.mod`、`Cargo.toml` |
| 测试运行器 | dev 依赖、`test` 脚本、既有测试文件 |
| 既有 linter | `eslint.config.*`、`biome.json`、`.ruff.toml` |
| 今天的覆盖率 | `coverage/` 输出；或由主代理用 terminal 跑一次测试套件 |
| CI | `.github/workflows/`、`.gitlab-ci.yml` |
| Agent 记忆 | `MEMORY.md`、`/var/minis/skills/` 下的技能目录 |

用两行汇报你发现的东西，然后只问剩下的。

### Step 2: 四个问题，每个都带默认值

遵循 `interview-me` 的"一次只问一个问题"纪律，只改一处：这里的每个问题都有默认值，所以"我不知道"是一个完整答案，仍能产出一份可用的配置。

```
Q1: 除 floor 之外，这些里你想强制哪些？
    (a) 新代码的测试覆盖率
    (b) 安全扫描
    (c) 性能预算
    (d) 无障碍
    (e) 架构边界
GUESS: (a) 和 (b) —— 你已经有测试运行器，而且你在处理用户输入。
DEFAULT if unsure: (a) 和 (b)。
说明每个选项的代价：(c) 和 (d) 需要一个可访问的 URL，(e) 需要写一份规则文件。
```

```
Q2: 当某个检查在 agent 干活途中失败时，应该拦下还是警告？
GUESS: 拦下。你在无人值守地跑 agent，没人读的警告等于没有警告。
DEFAULT if unsure: floor 上拦下，其余一切头两周先警告。
```

```
Q3: 你心里有目标数字，还是让我测出你今天的水平并守住那条线？
GUESS: 测量。多数团队没有数字，而编造出来的数字会被无视。
DEFAULT if unsure: 测量并守住。见下面的"棘轮（Ratchets）"。
```

```
Q4: 在 agent 把手头的活交回之前，你能容忍的最慢检查是多久？
GUESS: 大约 90 秒。再长你就不会去跑了。
DEFAULT if unsure: 任务结束时 90 秒。
```

停在四个。十二个问题的入口调研会产出一份没人理解的配置，和一个后悔开始的用户。

### Step 3: 写 CONSTRAINTS.md

一个文件，放在仓库根。任何 harness 上的任何 agent 都能读它，而它的改动会出现在审查里——那才是它该出现的地方。

```markdown
# Constraints

Last reviewed: 2026-08-08 by @addy

## Floor (always enforced, no setup required)

- No new suppression comments: `@ts-ignore`, `eslint-disable`, `# noqa`, `# type: ignore`
- No unimplemented stubs: `throw new Error("Not implemented")`, empty `catch {}`
- No skipped or deleted tests without a reason in the commit message
- No secrets in source
- This file does not get weakened to make a change pass

## Enforced with numbers

| Dimension | Rule | Checked by | Runs at |
|-----------|------|-----------|---------|
| Types | Zero type errors | `tsc --noEmit` | 每次 implementation 交回后 |
| Lint | Zero errors from our config | `biome check` | 每次 implementation 交回后 |
| Secrets | No secrets in source | `gitleaks detect --redact` | 每次 implementation 交回后 |
| Coverage | Changed lines ≥ 80% covered | `vitest run --coverage` + git diff | 任务结束 |
| Security: code | No high findings | `semgrep scan --config p/default` | 任务结束 |
| Security: deps | Nothing at high or above | `osv-scanner scan source -r .` | 任务结束 |
| Accessibility | Zero critical or serious | `axe $PREVIEW_URL --tags wcag2a,wcag2aa,wcag21aa` | preview 服务起好后 |
| Performance | LCP ≤ 2500ms, CLS ≤ 0.1 | `lighthouse $PREVIEW_URL --output=json` | preview 服务起好后 |

Every row names the command that produces the verdict. A dimension with a
number and no command in this column is an aspiration, not a constraint.

## Measured, not yet enforced

| Metric | Today | Direction |
|--------|-------|-----------|
| Project coverage | 62.4% | must not fall |
| Bundle size (main) | 184 kB | must not grow |

## Exceptions

| ID | Rule | Path | Reason | Owner | Expires |
|----|------|------|--------|-------|---------|
| W1 | `no-explicit-any` | `src/legacy/**` | Rewrite tracked in ENG-441 | @addy | 2026-11-01 |
```

然后在 `MEMORY.md` 里加一行：`写代码前先读 CONSTRAINTS.md，不要为了让改动通过而削弱它。`（Eta 用 `MEMORY.md` 与当轮上下文，没有 `CLAUDE.md` / `AGENTS.md`。）

### Step 4: 为每个维度装上它需要的东西

> **Eta 说明：** 下面表格里的安装、配置和运行命令**全部由主代理用 terminal 执行**，不由任何子代理执行；这些工具也不是本技能每次应用时的必做安装项，只在对应维度确实需要时才装。

选一个维度就意味着装一个东西。别让用户拿到一个数字却没有机制，也别在已有事实标准检查器时自己发明一个——这些工具被列出来，是因为它们的规则格式与阈值正是生态里其余一切所对标的东西，所以团队既有的配置能继续用。

| 维度 | 工具 | 安装 | 运行 | 以什么为门 |
|-----------|------|---------|-----|-|
| 类型（TS） | tsc | 已就位 | `tsc --noEmit` | 任何错误 |
| 类型（Python） | mypy | `pip install mypy` | `mypy .` | 任何错误 |
| Lint | 你既有的配置 | 已就位 | `eslint .` / `biome check` / `ruff check` | 任何错误 |
| 覆盖率（JS） | 你的测试运行器 | 已就位 | `vitest run --coverage`（或 `jest --coverage`） | 变更行的覆盖率 |
| 覆盖率（Python） | pytest-cov | `pip install pytest-cov` | `pytest --cov --cov-report=lcov` | 同上 |
| 安全：代码 | Semgrep | `pipx install semgrep` | `semgrep scan --config p/default --config p/owasp-top-ten` | 任何 high 发现 |
| 安全：密钥 | gitleaks | 按官方文档安装（本机为 Linux Debian，无 brew） | `gitleaks detect --redact --no-banner` | 任何发现 |
| 安全：依赖 | osv-scanner | 按官方文档安装（本机为 Linux Debian，无 brew） | `osv-scanner scan source -r .` | high 及以上 |
| 性能：页面 | Lighthouse | `npm i -D lighthouse` | `lighthouse $URL --output=json --quiet` | LCP、CLS、性能分 |
| 性能：包体 | size-limit | `npm i -D size-limit` | `size-limit --json` | 每个 entry 的字节预算 |
| 无障碍 | axe-core | `npm i -D @axe-core/cli` | `axe $URL --tags wcag2a,wcag2aa,wcag21aa` | 零 critical 或 serious |
| 架构 | dependency-cruiser | `npm i -D dependency-cruiser` | `depcruise --validate src` | 任何违规 |
| 断言质量 | Stryker | `npm i -D @stryker-mutator/core` | `stryker run --mutate <changed files>` | 变异分数 |

有五件事你若跳过就会被反咬：

1. **gitleaks 的 `--redact` 不是可选项。** 没有它，匹配到的密钥会落进 agent 的对话记录里，这就是一个泄露的 key 最后进入日志、摘要或提交信息的方式。只报规则与位置，绝不报值。
2. **Lighthouse 和 axe 需要一个 URL。** 它们只能对着一个运行中的应用工作，所以属于在 preview 服务或先由主代理用 terminal 起好的本地服务上运行的阶段。如果项目没有可打的 URL——一个 CLI、一个库、一个桌面应用——就说清楚并砍掉这个维度，而不是发明一个跑不起来的检查。
3. **把昂贵的检查限定到 diff。** 对整个仓库跑 `stryker run --mutate` 要几个小时，然后会被关掉；只在改动触及的文件上跑不到一分钟。Semgrep 同理，它接受一个路径列表。
4. **覆盖率不需要跑第二遍测试。** 读取你的套件已经写出的 lcov，再与 `git diff` 求交。为了拿一个数字而把套件跑两遍，是让人讨厌这套东西的最快方式。
5. **Semgrep 的 registry 规则可以免费运行；在再分发它们之前先看许可证。** 如果你的法务在意，`opengrep` 是一个同规则格式、同 JSON 输出的即插即用分叉。

把每一项加进项目自己的脚本，让它脱离 agent 也能复现：

```json
{
  "scripts": {
    "check:fast": "tsc --noEmit && eslint . && gitleaks detect --redact --no-banner",
    "check:task": "npm run check:fast && vitest run --coverage",
    "check:full": "npm run check:task && semgrep scan --config p/default && osv-scanner scan source -r ."
  }
}
```

那张映射比工具本身更重要。`check:fast` 是每次实现交回后跑的，`check:task` 是当 agent 认为它做完时跑的，`check:full` 是更完整的一档。这些命令由**主代理用 terminal 执行**，而不是由子代理执行。

命令现在活在两个地方——`CONSTRAINTS.md` 的 `Checked by` 列，和这些脚本。`CONSTRAINTS.md` 是权威来源：它在每条命令旁带着理由，而且会出现在审查里。脚本是必须镜像它的便利包装，不是第二个真相来源；如果它们漂移，以文件为准。

### Step 5: 接到生命周期上

最大的错误是在每个地方都跑所有东西。一个拖住 agent 的检查会被关掉，而一个被人们关掉的门比没有门更糟，因为标准线看上去还在。

| 阶段 | 触发者 | 跑什么 | 预算 |
|-------|---------|-----------|--------|
| 实现 | implementation 子代理（平台自动为它建隔离 worktree） | 只写文件；子代理不跑检查 | —— |
| 交回 | 每次 implementation 子代理交回后，主代理用 terminal 执行 | 类型、lint、密钥、floor，仅限变更文件 | 几秒之内 |
| 验证 | 主代理用 terminal 执行 | 相关测试、变更行的覆盖率 | 90 秒之内 |
| 审查 | review 子代理在给定 workspace 上只读检查 | 全部，外加下面的 guard | 数分钟 |
| 收尾 | 主代理在 `manage_agent_workspace` 合并前核验 | 方向检查、无回归 | —— |

**检查的落点（Eta 现实）：** Eta 没有 hook、没有 CI，所以检查从原文的"编辑后立即跑"改为"**每次 implementation 子代理交回后，由主代理用 terminal 执行**"。这是唯一落点，接受节奏比原文慢一档。

两条让这变得可忍受的规则：

1. **限定到 diff。** 检查这次改动触及的行，而不是整个仓库。变更行的覆盖率是 agent 能推动的数字；项目覆盖率是它继承来的数字。
2. **成本决定位置。** 任何超过几秒的东西都移出交回环节。对整个仓库做变异测试要几个小时；对改动触及的文件做不到一分钟，这就是人们会跑的检查和不会跑的检查之间的差别。

### Step 6: 守住标准线本身

有人会指出：如果 agent 既写代码又写检查，那这些检查什么也证明不了。一半对，而且值得围绕它做工程。

Agent 不会精心设计巧妙的漏洞。它们撞上一个红灯检查，然后走最便宜的路到绿灯。在审查阶段，盯住 diff 里的这五种动作：

1. **阈值被移动。** 预算被调低、严重度被降级、某个检查从快档里被移除。把 `CONSTRAINTS.md` 与它在分支点时的状态比较。
2. **测试被做简单。** 加了 `.skip`、删了测试文件、从保留的测试里抽掉了断言。
3. **检查器被静音。** 新增的 `@ts-ignore` 或 `eslint-disable`。有四类抑制特别值得注意，因为它们关掉的是你正依赖的检查：`istanbul ignore` 把代码从覆盖率里丢掉而不是去测试它，`Stryker disable` 藏起一个存活的变异体，`nosemgrep` 和 `gitleaks:allow` 对安全发现做同样的事。
4. **工作未完成。** 一个会抛异常的 stub、一个把失败变成沉默的空 `catch`、一个立在实现本该所在处的 `TODO`。
5. **出现了一个例外。** Exceptions 表里多了一行没人讨论过的记录。

这些都不需要 `git diff` 之外的工具。收紧标准线应当静默；放松它应当出声。`git diff` 由主代理用 terminal 执行。

与编号维度不同，floor 没有它自己的事实标准工具，所以被要求执行它的 agent 往往从零写一个检查器，两个 agent 写出两个不同的。Eta 里没有 hook、没有 CI，所以 floor 由**主代理在每次 implementation 子代理交回后用 terminal（`git diff` 等）执行**。执行时守住这份契约：

- **输入：** merge base 到工作树之间的 diff（新增行*与*删除行，加上未跟踪文件）。只读 `git diff` 的守卫会漏掉新文件和已暂存但未提交的工作。
- **检测 Step 6 的五种动作：** `CONSTRAINTS.md` 里被削弱的阈值；被做简单的测试（`.skip`、被删的测试文件、从保留下来的测试里抽掉的断言）；被静音的检查器（新加的抑制注释）；未完成的工作（stub 或空 `catch`）；新出现的 Exceptions 行。
- **成败判定：** 干净 / 至少一处 floor 违规（拦下改动）/ 无法运行（没有 merge base、不是 git 仓库）。绝不允许"无法运行"被读成"干净"。
- **只报规则与位置，绝不报匹配到的密文值。** 遮蔽不是可选项（见 Step 4）。
- **收紧要静默，放松要出声：** 只浮现降低标准线的动作。

**并非所有检查都同样循环。** 用一个问题给它们排序：agent 能不能靠写不能工作的代码让它通过？

- **外部** — axe-core 编码了 WCAG，`osv-scanner` 读漏洞数据库，Lighthouse 测量真实浏览器。agent 无法跟这些争辩。
- **项目** — 你的 lint 规则、你的分层边界。一个人类拥有那个文件。
- **套件** — 你自己的测试。最有用，也是唯一真正循环的一种。

一条完全由第三类构成的标准线，价值低于一条掺了外部意见的标准线。检查至少存在一个外部约束。

### Step 7: 棘轮，当你没有数字时

在一个 62% 覆盖率的代码库上设 80%，你会永远得到一个红灯构建，然后一个学会无视红灯构建的团队。

替代方案不需要任何决策：记录你现在的水平，然后拒绝变差。把今天的数字和一个方向放进"Measured, not yet enforced"表。每个检查都与记录值比较，而不是与一个愿景比较。当数字改善，更新它；当它下降，那就是发现。

这也回答了一个关于训练的合理反驳。模型因通过测试而被奖励，而测试你能在几秒内评估。架构腐化在几个月里显现，永远进不了权重。棘轮就是缺失的那个惩罚，写在构建能看到的地方。

## 合理的默认值（Sane Defaults）

当用户没有意见时，用这些。它们被选成大多数代码库在第一天就能满足。

| 约束 | 默认值 | 为什么是这个数字 |
|------------|---------|-----------------|
| 变更行的覆盖率 | ≥ 80% | 高到足以逼出一个测试，低到允许一行配置 |
| 项目覆盖率 | 今天的值，不得下降 | 采纳它无需任何争论 |
| 变异分数（若使用） | 起步 ≥ 60% | 对于一个从未变异过的套件很典型；80% 是成熟 |
| 依赖漏洞 | high 及以上不得存在 | 低于此的大多是噪音 |
| LCP | ≤ 2500 ms | Core Web Vitals 的"good"阈值 |
| CLS | ≤ 0.1 | 同上 |
| 无障碍 | 零 critical 或 serious 的 axe 违规 | moderate 与 minor 常常有争议 |
| 例外存活期 | 90 天 | 长到能规划修复，短到还记得住 |
| 棘轮容忍度 | 0.5% | 当一个无关文件移动数字时吸收漂移 |

把数字和理由一起陈述。一个没有理由的阈值会被下一个撞上它的人删掉。

## 升级路径（Escalation Path）

约束在三个牙齿等级上工作。从第一级开始。

1. **仅成文。** `CONSTRAINTS.md` 存在，agent 读它。不花成本，抓住诚实的错误，依赖 agent 遵守。
2. **脚本化。** 一个 `npm run check`（或 `make check`）跑快检查。Eta 没有 post-edit hook、没有 CI，所以它由主代理在每次 implementation 子代理交回后用 terminal 执行。确定性强，不引入新依赖。
3. **工具支撑。** 一个专门运行器处理 diff 范围、预算、棘轮和 guard 检查。当配置长到超过一个 shell 脚本时使用。

多数项目应当停在 2。当你维护的跑检查的 shell 超过大约三十行时，移到 3。

**第一次运行可以只有 floor。** floor guard 是仅 diff 的，且不需要安装，所以你可以第一天就强制 floor，再随着逐个安装工具加入编号维度，而不是在第一次提交被保护之前就把每个检查器都立起来。机器级安装的安全工具可以作为任务结束阶段的检查。在 `Runs at` 列里声明每个维度在哪里跑。

## 常见合理化

| 借口 | 现实 |
|--------|---------|
| "等代码稳定了再加约束" | 代码会围绕着它流动时被允许的东西稳定下来 |
| "测试就是约束" | 你自己写的测试只证明你同意你自己；它们对新代码的覆盖率、依赖风险或包体增长什么也没说 |
| "我们达不到 80% 覆盖率" | 那就别设 80%。设今天的数字并守住它 |
| "这会拖慢 agent" | 只有你把慢检查放进快环节时才会。那是布置错误，不是反对约束的论据 |
| "我会记住我们的标准是什么" | agent 记不住，而大部分代码是它在写 |
| "约束会挡住我们发布" | 一个带负责人和日期的例外会放你走。删掉约束会永远放所有人走 |

## 红旗（Red Flags）

如果你注意到以下情况，停下并重新考虑：

- 访谈超过四个问题，或产出了一份用户解释不清的配置
- 设了一个代码库今天达不到的预算，却没有达到它的计划
- 某个维度被写进 CONSTRAINTS.md，有数字，却没有工具在背后支撑
- 在已有事实标准检查器时手搓了一个检查器，于是团队既有的配置被无视
- 每个约束都由项目自己的测试套件检查，没有外部意见
- `CONSTRAINTS.md` 与那个正在失败的改动出现在同一个提交里
- 某个例外没有负责人，或过期时间在一年以上
- agent 提议放宽阈值，而不是修代码
- 慢检查落进了交回环节，已经有人开始传 `--no-verify`
- 自它写出来以后，没人打开过 `CONSTRAINTS.md`

## 验证

本技能被正确应用当且仅当：

- [ ] `CONSTRAINTS.md` 存在，且其中每个数字都有陈述出来的理由
- [ ] floor 被强制，且在当前代码库上无需改动即通过
- [ ] 用户选的每个维度都有一个今天能跑的工具和命令
- [ ] 每条约束都说明它在哪里跑，且快环节保持在几秒内
- [ ] 至少有一条约束是外部的（不由本项目自己的测试判定）
- [ ] 仅测量的指标记录了今天的值和一个方向
- [ ] 例外有负责人和过期日期
- [ ] `MEMORY.md` 指向该文件（或当轮上下文里已交代）
- [ ] 在当前分支上做一次试运行，不产生用户不认同的失败

## 另见（See Also）

- `verification-before-completion` — 试运行与验收证据的纪律：约束是否被满足，以验证证据为准
- `systematic-debugging` — 约束被违反时，按四阶段定位根因而不是猜修法
