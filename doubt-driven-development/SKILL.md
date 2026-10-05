---
name: doubt-driven-development
description: Subjects every non-trivial decision to a fresh-context adversarial review before it stands. Use when you want every assumption cross-examined before proceeding, when stress-testing a plan for hidden failure modes, when correctness matters more than speed, when working in unfamiliar code, when stakes are high (production auth, security-sensitive logic, a high-stakes migration, irreversible operations), or any time a confident output would be cheaper to verify now than to debug later. 在 Eta 里由主代理对非平凡决策做对抗式复审。
---

# 怀疑驱动开发（Doubt-Driven Development）

## 概述

自信的答案不等于正确的答案。长会话会不断累积上下文，在无人察觉的情况下把假设慢慢变成"事实"。怀疑驱动开发是一门纪律：在任何非平凡产出成立之前，先物化出一个**带着全新上下文的审查者**——它的偏向是**证伪**，而不是批准。

这不是收尾阶段的 review 裁定。review 是对已完成产物的判定；怀疑驱动是一种进行中的姿态——非平凡的决策在纠偏仍然廉价的时候就被交叉盘问。

## 何时使用

一个决策只要满足以下任一条，就是**非平凡**：

- 它引入或修改分支逻辑
- 它跨越模块或服务边界
- 它断言了类型系统或编译器无法验证的性质（线程安全、幂等、顺序、不变量）
- 它的正确性依赖未来读者看不到的上下文
- 它的影响范围不可逆（生产部署、数据迁移、公共 API 变更）

在以下情况应用本技能：

- 在不确定下即将做架构决策
- 即将提交非平凡代码
- 即将断言一个非显然事实（"这是安全的""这能扩展""这符合规格"）
- 在不完全理解的代码里工作

**何时不要用：**

- 机械操作（改名、格式化、移动文件）
- 遵循清晰无歧义的用户指令
- 阅读或总结既有代码
- 正确性显而易见的一行改动
- 纯工具操作（跑测试、列文件）
- 用户已明确要求速度优先于验证

如果对每一次敲键都怀疑，你什么都发布不了。本技能只适用于上文定义的非平凡决策。

## 加载与编排约束（Loading Constraints）

本技能为主代理（操盘手）设计，Step 3（DOUBT）会派发一个全新上下文的审查子代理。

- 派发方式：`delegate_task(role="review", project=<git 仓库根>, workspace_id=<该 workspace_id>, context=<对抗式提示 + ARTIFACT + CONTRACT>)`。review 子代理是只读的，不能执行 shell、GUI 或浏览器；它只能 `read_file` / `list_directory`。
- **任何子代理都不得再派发子代理。** 因此子代理无法执行本技能 Step 3。若你在子代理上下文里，正确做法是把"无法嵌套运行 doubt"这件事显式上报给主代理，由主代理处理；降级方案（见下）只作最后手段。
- 需要跑命令的环节（例如取 `git diff`、跑测试来产出证据）由**主代理用 terminal 执行**，再把结果交给 review 子代理作为 ARTIFACT。
- **未经用户授权，不得把任何 artifact 或材料外发。** Eta 没有模型选择，也没有外部复核通道；doubt 复审只在本地由 review 子代理完成。任何把材料送往外部的动作都视为外发，未经用户明确授权不得进行。这条安全约定是硬性的。

**授权范围（按任务一次）：** 每个任务在开始时向用户申请**一次**授权，范围涵盖该任务内允许派发 review 子代理做对抗式复审、以及覆盖哪些 artifact；并在最终输出里**显式声明授权范围**。不要在同一个任务的每个 cycle 里反复询问用户——批量委派时那会让审查轮次翻倍。跨任务或扩大范围时需要重新取得授权。

**降级方案（仅在无法升到主代理时使用）：** 把 ARTIFACT + CONTRACT 重写成一个全新的自我提示，与先前推理之间画一道硬性心智隔离，然后走 Step 1–5。这**不是**全新上下文复审（你带着自己的上下文），因此必须把结果标记为降级，并且只要用户可达就优先选择上报。

## 流程

应用本技能时抄下这份清单：

```
Doubt cycle:
- [ ] Step 1: CLAIM — 写下主张 + 为什么重要
- [ ] Step 2: EXTRACT — 隔离出 artifact + contract，剥离推理过程
- [ ] Step 3: DOUBT — 以对抗式提示派发全新上下文的 review 子代理
- [ ] Step 4: RECONCILE — 把每条发现对照 artifact 文本分类
- [ ] Step 5: STOP — 满足停止条件（琐碎发现、3 个 cycle，或用户覆盖）
```

### Step 1: CLAIM — 浮现站得住的东西

用两三行点出这个决策：

```
CLAIM: "The new caching layer is thread-safe under the
        read-heavy workload described in the spec."
WHY THIS MATTERS: a race here corrupts user data and is
                  hard to detect in QA.
```

如果你无法把主张写得这么紧凑，那你拥有的是一种直觉，不是一个决策。先把它浮现出来，再审视它。

### Step 2: EXTRACT — 最小可审查单元

全新上下文的审查者需要的是 **artifact** 和 **contract**，不是你的旅程。

- 代码：diff 或那个函数——不是整个文件
- 决策：3–5 句话的提案，外加它必须满足的约束
- 断言：主张本身，加上据称支持它的证据（与 Step 1 的 CLAIM 块分开保留，后者是操盘手受审视的假设）

剥离你的推理。如果你递过去的是结论，你收回来的就是对结论的确认。这个单元必须小到审查者一次阅读就能在脑中握住——如果是一个 500 行的 PR，先分解。

### Step 3: DOUBT — 派发全新上下文的审查者

审查者的提示**必须是对抗式的**。框架决定答案。

```
Adversarial review. Find what is wrong with this artifact.
Assume the author is overconfident. Look for:
- Unstated assumptions
- Edge cases not handled
- Hidden coupling or shared state
- Ways the contract could be violated
- Existing conventions this might break
- Failure modes under unexpected input

Do NOT validate. Do NOT summarize. Find issues, or state
explicitly that you cannot find any after thorough examination.

ARTIFACT: <paste artifact>
CONTRACT: <paste contract>
```

**只传 ARTIFACT + CONTRACT，不要传 CLAIM。** 把你的结论交给审查者会诱导它同意你。审查者必须独立判断 artifact 是否满足 contract。

派发时把上面的对抗式提示**逐字**放进 `delegate_task` 的 `context`，并要求 review 子代理按"只列问题"的形状回复。**对抗式提示优先于审查者默认的回复形状。** 有些审查模板被写成给出平衡的裁定（同时列优点与缺点）；怀疑驱动需要的是只输出问题。因此把对抗式提示逐字粘进 context 以覆盖该默认形状；只有当某个审查模板的回复形状无法被这段提示覆盖时，才改派一个 review 子代理并明确要求只列问题。

### Step 4: RECONCILE — 把发现折回来

审查者的输出是数据，不是裁定。**你仍然是操盘手。** 在分类之前，把每条发现重新对照 artifact 文本读一遍——给审查者盖上橡皮图章，和忽视它，是同一种失败模式。

对每条发现，按这个**优先级顺序**分类（第一个匹配的类别胜出）：

1. **Contract 误读** — 审查者之所以标记它，具体是因为你提供的 CONTRACT 不清楚或不完整。先修 contract，下一 cycle 重新分类。
2. **有效且可行动** — 真实问题，需要改动 artifact。改它，重新循环。
3. **有效取舍** — 问题是真的，但修复成本超过接受的成本。把取舍显式记录下来，让用户看到。
4. **噪音** — 审查者标记的东西在它没有的上下文下其实是对的。记下，继续，并问一句：把这个上下文加进 contract 是否能避免这次误报？

全新上下文的审查者可能因为缺少上下文而犯错。不要仅仅因为它是"全新"的就盲从。

### Step 5: STOP — 有界循环，不是递归

在以下任一情况停止：

- 下一次迭代只返回琐碎的或已考虑过的发现，**或**
- 完成 3 个 cycle（上报给用户，不要独自硬磨第四个），**或**
- 用户明确说"发布吧"

如果 3 个 cycle 后审查者仍在给出实质问题，这个 artifact 可能还没准备好。把这点上报给用户——三个未解决的 cycle 是关于 artifact 的信息，不是继续循环的理由。

如果因为 artifact 很大而觉得"3 个 cycle 明显不够"：那说明 artifact 太大了——回到 Step 2 分解它。不要抬高上限。

## 常见合理化

| 合理化 | 现实 |
|---|---|
| "我很确信，跳过 doubt 吧" | 在新问题上，自信与正确性关联很弱。确信的时刻正是盲点藏身之处。 |
| "派发一个审查者很贵" | 在生产里调试一个错误提交更贵。检查是有界的；bug 不是。 |
| "审查者只会挑刺" | 只有范围失控时才这样。把提示约束到"在 contract 下会让实现失败的问题"。 |
| "我到收尾时才做 doubt" | 收尾的 review 是最后一道门。怀疑驱动在方向仍可廉价纠偏时就抓住错误方向。到收尾阶段已经太晚。 |
| "如果每一步都怀疑，我什么都发布不了" | 本技能只适用于非平凡决策，不是每一次敲键。重读"何时不要用"。 |
| "两个意见总比一个好" | 当第二个意见上下文更少、只产生噪音时并非如此。要归类，不要盲从。 |
| "审查者不同意，所以我错了" | 审查者缺少你的上下文——分歧是信息，不是裁定。重读 artifact，分类，再决定。 |

## 红旗（Red Flags）

- 为一行改名或格式调整就派发全新上下文的审查者
- 把审查者输出当作权威，而不重读 artifact 文本
- 超过 3 个 cycle 还不向用户上报
- 用"这好吗？"而不是"找问题"去提示审查者
- 在高风险决策上因时间压力跳过 doubt
- 对未改动的 artifact 反复派生全新上下文审查者（你会得到同样的发现；你在拖延）
- **Doubt theater（可核查信号）**：在 2 个或以上产生了实质发现的 cycle 里，零条发现被归类为 actionable。你在验证，不是怀疑。停下并上报。
- 只在提交之后才怀疑——那是 review，不是怀疑驱动开发
- 在同一个任务的每个 cycle 里反复向用户索要授权（应按任务授权一次）
- 把 contract 从审查者的输入里剥掉
- 把 CLAIM 传给审查者（诱导同意）
- 把 artifact 或材料外发给任何外部通道而未获用户授权

## 与其他技能的交互

- **`systematic-debugging`**：当怀疑被证实——审查者或对抗复审浮现出真实的失效模式——切入它做根因定位与修复。怀疑驱动负责发现，系统化调试负责定位与修。
- 本技能由主代理执行编排；子代理不得调用其他子代理（见上文"加载与编排约束"）。

## 验证

应用怀疑驱动开发之后：

- [ ] 每个非平凡决策（按上文定义）在成立前都被显式命名成 CLAIM
- [ ] 每个非平凡 artifact 至少经过一次全新上下文复审（TDD 的 RED 步产生的失败测试，对行为类主张而言即满足，见"与其他技能的交互"）
- [ ] 审查者收到的是 ARTIFACT + CONTRACT——不是 CLAIM，也不是你的推理
- [ ] 审查者的提示是对抗式的（"找问题"），不是验证式的（"这好吗"）
- [ ] 发现被对照 artifact 文本分类（不是橡皮图章），优先级为：contract 误读 / actionable / trade-off / noise
- [ ] 满足了某个停止条件（琐碎发现、3 个 cycle，或用户覆盖）
- [ ] 派发 review 子代理的授权是**按任务申请一次**的，且授权范围已在最终输出里显式声明
- [ ] 未在未获用户授权的情况下把任何材料外发
