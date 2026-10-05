---
name: writing-plans
description: Use when you have a spec or requirements for a multi-step task, before touching code（在动代码之前，为多步骤任务编写实现计划时使用）
---

# Writing Plans（编写实现计划）

## Overview（概述）

为一个从未见过这个代码库、也从未见过这份 spec 的实现者写实现计划。假设他们一旦知道确切的接口和确切的测试，就能用项目语言写出地道代码，并且凡是计划留白之处他们都会做出合理选择。他们无法知道的是你的决定：改哪些文件、用哪些名字和签名、spec 里的哪些取值、哪些测试能证明每个任务。把这些写下来。把整份计划拆成一口一个的小任务。DRY。YAGNI。TDD。频繁提交。

**开始时声明：** "I'm using the writing-plans skill to create the implementation plan."

**Context（上下文）：** 在 Eta 里，计划由主代理派给 implementation 子代理执行；隔离工作区由平台自动为该子代理创建，主代理随后用 `manage_agent_workspace` 处理。计划中不要写入任何让子代理自建/自删 git worktree 的步骤，也不要把"保留分支、什么都不做"当作合法终点。

**计划保存到：** `docs/plans/YYYY-MM-DD-<feature-name>.md`
- （用户对计划存放位置的偏好覆盖此默认值）

## Scope Check（范围检查）

如果 spec 覆盖多个相互独立的子系统，它应该在头脑风暴阶段就被拆成子项目 spec。如果没有，建议拆成多份计划——每个子系统一份。每份计划都应能独立产出可运行、可测试的软件。

## File Structure（文件结构）

在定义任务之前，先画出将创建或修改哪些文件、每个文件负责什么。分解决策在这里被锁定。

- 设计边界清晰、接口明确的单元。每个文件应只有一个明确职责。
- 你对自己能一次性放进上下文的代码推理得最好，文件聚焦时你的编辑也更可靠。优先选择更小、更聚焦的文件，而不是职责过多的大文件。
- 一起改动的文件应放在一起。按职责拆分，而不是按技术分层拆分。
- 在既有代码库中，遵循既定模式。如果代码库使用大文件，不要擅自重构——但如果你要修改的文件已经臃肿到难以维护，把拆分写进计划是合理的。

这个结构决定任务分解。每个任务都应产出可独立理解、自包含的改动。

## Task Right-Sizing（任务粒度）

任务是能自带一轮测试周期、并且值得一个全新审查者把关的最小单元。划定任务边界时：把搭建、配置、脚手架和文档步骤并入其交付物需要它们的那个任务；只有在审查者能合理地否决一个任务却批准相邻任务时，才拆分。每个任务以一个可独立测试的交付物结束。

## Step Granularity（步骤粒度）

**每个步骤是一个动作，带一个可检查的结果：**
- "写失败的测试" —— 一个步骤
- "运行它，确认它失败" —— 一个步骤
- "写最小代码让测试通过" —— 一个步骤
- "运行测试，确认通过" —— 一个步骤
- "提交" —— 一个步骤

**平台约定（Eta）：** 需要 shell 的步骤（运行测试、`grep`、`git add`/`git commit` 等）不在子代理里执行，而是写明"由主代理用 `terminal(environment=linux)` 执行"；只读检查（读文件、看 diff 内容）可写成"由 review 子代理在给定 workspace 上只读检查"。

## Plan Document Header（计划文档头部）

**每份计划都必须以这个头部开始：**

````markdown
# [Feature Name] Implementation Plan

> **For agentic workers:** 由主代理把每个任务派给 implementation 子代理（project=仓库根），任务完成后由 review 子代理在同一个 workspace 上核验，主代理再用 `manage_agent_workspace` 合并；主代理不自己下场写代码。参见本目录下的兄弟技能 subagent-driven-development。步骤用复选框（`- [ ]`）语法跟踪。

**Goal:** [一句话说明要构建什么]

**Architecture:** [2-3 句说明方法]

**Tech Stack:** [关键技术/库]

**Spec:** [本计划实现的 spec/设计文档路径 —— 计划是从 spec 论证出来的，所以 spec 随计划一起走；执行者两份都读]

## Global Constraints（全局约束）

[spec 的项目级要求 —— 版本下限、依赖限制、命名与文案规则、平台要求 —— 每行一条，取值从 spec 逐字照抄。每个任务的要求都隐含包含本节。]

## Review Focus（审查重点）

[spec 暗示、但没有任何任务的测试覆盖到的、最可能坑到使用者的五类输入或失败模式 —— 每行一条，写明输入或条件以及一个理性人会预期的行为，最可能的放前面。spec 是一份愿景文档：它说明软件必须做什么，而不是它会遇到的一切；它对某个输入保持沉默，不等于允许该输入把程序弄崩。把这份清单在这里写一次，写的时候把 spec 摆在面前。然后，为每一行，把钉住它的测试加进拥有该代码的那个任务，用该任务自己的步骤风格。]

---
````

## Task Structure（任务结构）

````markdown
### Task N: [Component Name]

**Files:**
- Create: `exact/path/to/file.py`
- Modify: `exact/path/to/existing.py:123-145`
- Test: `tests/exact/path/to/test.py`

**Interfaces:**
- Consumes: [本任务用到前面任务的什么 —— 精确签名]
- Produces: [后面任务依赖什么 —— 精确函数名、参数与返回类型。一个任务的实现者只看得见自己的任务；这个块就是他们了解相邻任务所用名字与类型的地方。]

- [ ] **Step 1: 写失败的测试**

```python
def test_specific_behavior():
    result = function(input)
    assert result == expected
```

- [ ] **Step 2: 运行测试，确认它失败**

由主代理用 `terminal(environment=linux)` 执行：`pytest tests/path/test.py::test_name -v`
预期：FAIL，报 "function not defined"

- [ ] **Step 3: 在 `exact/path/to/file.py` 实现 `function(input: InputType) -> ResultType`**

当签名和测试还留有选择空间时，用一行说明方法（调哪个库函数、用哪种数据结构）；只有算法无法由签名和测试确定时才给代码块。

- [ ] **Step 4: 运行测试，确认它通过**

由主代理用 `terminal(environment=linux)` 执行：`pytest tests/path/test.py::test_name -v`
预期：PASS

- [ ] **Step 5: 提交**

由主代理用 `terminal(environment=linux)` 执行：
```bash
git add tests/path/test.py src/path/file.py
git commit -m "feat: add specific feature"
```
（若用户希望自行处理版本控制，也可只输出补丁与提交信息，由用户在需要时处理。）
````

## What a Step Contains（一个步骤包含什么）

当实现者能据此写出恰好一件合理的事时，这个步骤就完成了。这就是全部要求：无歧义，而非完备。每一类步骤只携带让它无歧义所需的东西，不多不少：

- **测试步骤：** 测试的名字和它的断言，以代码形式给出，里面带上 spec 的确切取值。
- **代码步骤：** 确切的签名（名字、参数、返回类型）、所在文件、以及 spec 钉死的具体取值。实现者写函数体。只有签名和测试无法确定算法、或 spec 固定了逐字文案时，才给出函数体。
- **验证步骤：** 要运行的命令，以及代表通过的输出。命令由主代理用 `terminal(environment=linux)` 执行，输出再喂回实现者/审查者。
- **对另一个任务的引用：** 那个任务的 Interfaces 块说明了要用什么；计划不重复那个任务的代码。

计划是实现者无法独自做出的那组决定。比它所描述的代码还长的计划，等于把代码写了一遍。不决定任何东西的行（"TBD"、"处理边界情况"、"加适当的校验"、"为上述写测试"、某个没有任何任务定义的类型或函数）是相反的失败，自审要抓住这两种。

## Self-Review（自审）

写完完整计划后，用全新的眼光重读 spec，把计划对照它检查。这是你自己跑的一份清单 —— 不是派子代理去做。

**1. Spec 覆盖：** 浏览 spec 的每一节/每项要求。你能指出一个实现它的任务吗？列出所有缺口。

**2. 步骤扫描：** 每个步骤都必须让实现者能写出恰好一件合理的事，且不能超过这一点：不决定任何东西的行是缺口，签名和测试已经确定的函数体是誊写。两者都要修。

**3. 类型一致：** 你在后面任务里用的类型、方法签名和属性名，是否与你在前面任务里定义的一致？Task 3 里叫 `clearLayers()`、Task 7 里叫 `clearFullLayers()` 的函数是一个 bug。

**4. Review Focus：** 对 spec 暗示的每一类输入或失败模式，是否有一个任务的测试覆盖它？最可能坑到使用者的五个未覆盖项放进 Review Focus 一节，那里的每一行都要把它的测试加进拥有该代码的任务。一节留空意味着你查过且没有发现，而不是你跳过了检查。

**5. 比例：** 比较计划长度与 spec 长度。一份比它实现的 spec 长好几倍的计划，是程序的誊写，而不是计划。如果代码块占了文档大部分，把函数体换成签名、测试名和断言，并检查每个步骤是否仍然无歧义。

如果发现问题，就地修掉。不必重新审查 —— 修完继续。如果你发现一条 spec 要求没有对应任务，补上那个任务。

## Execution Handoff（交接执行）

保存并自审计划后，在主代理的回复里把计划链接给用户阅读，请其审阅并确认。Eta 的默认执行方式是：由主代理把每个任务派给 implementation 子代理，任务完成后由 review 子代理在同一个 workspace 上核验，主代理再用 `manage_agent_workspace` 合并 —— 主代理不自己下场写代码，因为主代理亲自实现会失去每个任务的独立审查。

**默认（推荐）执行方式：**

**"Plan complete and saved to `docs/plans/<filename>.md`. Please review the plan. 默认按"implementation 子代理逐个任务实现 + 每个任务过 review gate，主代理最后合并"来做。Does the plan capture what you want?"**

把计划直接呈现在主代理的回复里，请用户在后续消息中确认；在用户确认之前不要开始实现。

**若用户明确要求主代理亲自实现：** 明确告知这会失去每个任务的独立审查，只有在该实现改动小、风险低时才可接受；随后由主代理实现，并在结束前安排一次 review 子代理做整体核验。
