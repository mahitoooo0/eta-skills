---
name: android-gui-automation
description: Android 界面自动化实战纪律：观察-动作-验证循环、节点引用与过期处理、文本输入方式选择、滚动与等待时机、失败恢复。任何需要在 App 界面上点击、输入、翻页的任务使用。
---

# Android GUI Automation

界面操作的核心是**观察-动作-验证循环**：不盲点、不重放过期引用、每步有依据。

## 基本循环

```
observe_screen（默认只读 UI 树，不带截图）
  → 定位唯一节点（text / content-desc / id）
  → tap_element（带这次观察的 observation_id）
  → 需要确认结果时再 observe
```

- **屏幕以视觉内容为主**（Canvas、地图、图片、二维码）或任务依赖颜色/布局时，才 `include_screenshot=true`，并保持 UI 树同时返回。
- UI 树被截断但节点语义够用时，先提高 `max_nodes`，不要急着截图。

## 显示模式：先判主屏还是副屏

Eta 可能在虚拟显示（副屏）上运行，两种模式的操作规则不同：

- **判别**：observe_screen 返回的 `ui_nodes` 为空、工具提示「仅截图模式，不支持节点操作」→ 当前在副屏。
- **副屏模式**（纯截图，无无障碍节点）：
  - 每次观察都带 `include_screenshot=true`——截图是唯一视觉依据，本技能「视觉内容为主才截图」的默认**反着来**。
  - 动作用坐标工具：点击/长按/滑动默认取**最近一次截图的像素坐标**（坐标来源不是当前截图时设置 `coordinate_space=screen`）。
  - 节点类工具（tap_element 等带 index/observation_id 的）不可用，识别到副屏后不要反复尝试。
  - 文本输入（replace_text/paste_text）在副屏未经充分验证：先小步试一次，失败就报告用户，不循环重试。
  - 失败恢复里「ACCESSIBILITY_* → 禁止坐标重放」**不适用于**副屏——坐标在副屏本来就是唯一路径。
- **主屏模式**：下面所有节点规则照旧执行。

无论哪种模式：滚动方向语义、点击成功后不例行观察、高危动作门禁（支付/发送/删除先确认）全部适用。

## 节点纪律

- 调用 tap_element/replace_text 等**必须带同一次观察的 observation_id**；不确定就重新观察。
- 工具返回 `ACTION_OUTCOME_UNKNOWN` 或 `DIRECTION_MISMATCH` → **先重新观察**，禁止直接重放动作。
- 目标无法唯一定位（重名、嵌套列表）→ 观察后用更精确的选择器，或先滚动让目标可见。

## 输入策略

| 场景 | 用法 |
|------|------|
| 短的精确文本 | replace_text |
| 长文本、中文、特殊字符 | **paste_text**（优先，不走输入法最稳） |
| 搜索框联想干扰 | 输入后观察建议列表是否遮挡，必要时回车确认 |

## 滚动与等待

- `scroll` 的方向是**要显示的内容方向**：看下面的内容就 scroll down。
- 点击/输入成功后**不例行观察**；只有这些情况才观察：要读屏幕信息、后续状态未知、节点过期、任务收尾要确认最终结果。
- 只在「后续操作依赖特定文本/应用出现」时用 wait_for_text / wait_for_package。
- 不依赖中间界面变化的连续操作，同一轮一并发出，不拆回合。

## 失败恢复

- `ACCESSIBILITY_UNAVAILABLE` / `ACCESSIBILITY_PROTECTION_UNAVAILABLE` / `ACCESSIBILITY_REPAIR_TIMEOUT` → 动作没执行，告知用户无障碍服务状态，**不要改用坐标或 shell 重放 GUI 动作**。
- 弹窗/广告遮挡 → 先观察识别弹层，关掉再继续原目标。
- 意外页面（登录、验证、权限请求）→ 停下来告诉用户，不自行输密码/点授权，除非用户已明确授权这一步。
- 更多场景见 `references/failure-recovery.md`。

## 高危动作门禁

支付、转账、发送消息/邮件、删除、下单、修改账号设置——执行前用 ask_user 向用户确认关键参数（对象、数量、金额），得到确认才执行。用户说「你看着办」只覆盖当前这一个问题，不扩权到后续操作。
