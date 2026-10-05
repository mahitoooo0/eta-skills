---
name: verification-before-completion
description: Use when about to claim work is complete, fixed, or passing, before committing or creating PRs - requires running verification commands and confirming output before making any success claims; evidence before assertions always（即将宣称工作完成、已修复或通过时使用：任何成功断言之前，先由主代理跑验证命令并确认输出，证据先于断言）
---

# Verification Before Completion（完成前验证）

## Overview（概述）

**核心原则：** 证据先于断言，永远如此。

**违反这条规则的字面，就是违反它的精神。**

## The Iron Law（铁律）

```
NO COMPLETION CLAIMS WITHOUT FRESH VERIFICATION EVIDENCE
没有全新的验证证据，就没有完成断言
```

如果主代理没有在本轮用 `terminal` 跑过验证命令，就不能宣称它通过。

## The Gate Function（闸门函数）

```
在做出任何状态判断或表达满意之前：

1. 识别（IDENTIFY）：哪条命令能证明这个断言？
2. 执行（RUN）：由主代理用 terminal(environment=linux) 跑完整命令（全新、完整）
3. 读取（READ）：完整输出，检查退出码，统计失败数
4. 核验（VERIFY）：输出是否支持断言？
   - 否：给出带证据的真实状态
   - 是：带着证据给出断言
5. 只有这样（ONLY THEN）：才做出断言

跳过任一步 = 撒谎，而不是验证
```

## Common Failures（常见失败）

| 断言 | 需要什么 | 不充分 |
|-------|----------|----------------|
| 测试通过 | 测试命令输出：0 失败 | 上一次运行、"应该会通过" |
| Linter 干净 | Linter 输出：0 错误 | 部分检查、外推 |
| 构建成功 | 构建命令：退出码 0 | Linter 通过、日志看着不错 |
| Bug 已修 | 测试原始症状：通过 | 代码改了、假设已修 |
| 回归测试有效 | 红-绿循环已验证 | 测试通过一次 |
| 子代理完成 | `manage_agent_workspace inspect` 显示变更 | 子代理自报 "success" |
| 需求满足 | 逐行清单 | 测试通过 |

## Red Flags - STOP（危险信号 —— 停下）

- 使用 "should"、"probably"、"seems to"（"应该"、"大概"、"似乎"）
- 在验证前表达满意（"Great!"、"Perfect!"、"Done!" 等）
- 未经验证就要提交或推送
- 相信子代理的成功自报
- 依赖部分验证
- 想着 "就这一次"
- 累了，想把活干完
- **任何暗示成功、却没有跑过验证的措辞**

## Rationalization Prevention（防止合理化）

| 借口 | 现实 |
|--------|---------|
| "现在应该能用了" | 去跑验证 |
| "我有信心" | 信心 ≠ 证据 |
| "就这一次" | 没有例外 |
| "Linter 过了" | Linter ≠ 编译器 |
| "子代理说成功了" | 独立验证 |
| "我累了" | 疲惫 ≠ 借口 |
| "部分检查够了" | 部分什么都证明不了 |
| "换个说法规则就不适用了" | 精神高于字面 |

## Key Patterns（关键模式）

**测试：**
```
✅ [主代理跑测试命令] [看到：34/34 通过] "所有测试通过"
❌ "现在应该通过了" / "看起来对"
```

**回归测试（TDD 红-绿）：**
```
✅ 写 → 跑（通过）→ 撤销修复 → 跑（必须失败）→ 恢复 → 跑（通过）
❌ "我写了回归测试"（没有红-绿验证）
```

**构建：**
```
✅ [主代理跑构建] [看到：退出码 0] "构建通过"
❌ "Linter 过了"（Linter 不检查编译）
```

**需求：**
```
✅ 重读计划 → 建清单 → 逐项验证 → 报告缺口或完成
❌ "测试通过，阶段完成"
```

**子代理派发：**
```
✅ 子代理报告成功 → 用 manage_agent_workspace inspect 检查变更 → 复验改动 → 报告真实状态
❌ 相信子代理的报告
```

**复验的两种正当做法：** 主代理用 `terminal` 重跑确认，或派一个 review 子代理在给定的 workspace 上只读复验（读文件、看 diff、按清单逐项核对）。两者都必须给出实际输出/实际变更作为证据。

## When To Apply（何时适用）

**始终在以下之前：**
- 任何形式的工作成功/完成断言
- 任何满意的表达
- 任何关于工作状态的正向陈述
- 提交、任务完成
- 进入下一个任务
- 派发子代理

**规则适用于：**
- 确切措辞
- 转述与同义词
- 成功的暗示
- 任何暗示完成/正确的沟通
