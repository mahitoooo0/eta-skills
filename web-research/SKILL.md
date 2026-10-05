---
name: web-research
description: 用离屏浏览器做网页调研与数据抓取：分页读正文、等动态渲染、脚本提取、UA/视口切换、登录态保持、文件下载到工作区。用户要求查资料、对比、监控网页、整理网页数据时使用。
---

# Web Research

用浏览器工具完成调研任务。原则：**读得到正文就不截图，抓得到数据就不数节点，关键结论要交叉验证**。

## 标准流程

```
navigate（完整 URL / 域名 / 搜索词）
  → get_readable（offset/max_chars 分页读 Markdown 正文）
  → 需要结构时 find_elements / get_backbone
  → 需要交互时再点击/输入/滚动
```

- 长文用 `get_readable` 的 offset/max_chars 分页读完，不断重新 navigate。
- 打不开或内容异常 → `wait_for_dom_stable` / `wait_for_selector` 等渲染完成再读。

## 动态页与数据提取

- 页面数据来自接口时，用 `execute_js` 提取内嵌 JSON（如 `__NEXT_DATA__`、window 变量）或读取网络层脚本暴露的数据，比解析 DOM 稳。
- 无限滚动列表：`scroll_and_collect` 边滚边收。
- 需要 JS 交互后才出现的内容：execute_js 触发后等 DOM 稳定。

## 身份与适配

| 场景 | 做法 |
|------|------|
| 移动版页面内容不全 | set_user_agent 切 desktop_chrome |
| 桌面版墙/验证 | 切 mobile_chrome 再试 |
| 布局错乱 | set_viewport 调整 |

## 登录态

- `get_cookies` 只返回 cookie 名和环境文件路径；在 Linux 会话中 source 对应 env 文件后用 `COOKIE_<NAME>` 变量。
- 登录状态可能跨任务共享，不承诺账户隔离；不把 cookie 明文写进回复、日志或文件。
- 需要登录的调研：让用户先在浏览器里登录一次，之后复用会话；不自动填账号密码。

## 下载与产出

- `fetch` 用当前页会话下载资源；文件落到 Linux 的 /workspace，完成后告知用户路径。
- 整理数据：小表直接在回答里给 Markdown；多页抓取结果写入 /workspace 下的文件（csv/md），报告路径与行数。
- 截图只在视觉信息必要时用（默认视口，整页加 full_page=true）。

## 质量红线

- 关键事实（价格、日期、数字、结论）至少两个独立来源交叉验证；单源就标注「单一来源，未交叉验证」。
- 区分「页面原文」与「你的推断」；引用时给出页面标题。
- 不绕过登录墙/付费墙/验证码；不擅自提交表单、下单、点赞——用户明确要求时也先确认关键参数。
- 付费/计费操作（下载付费资源等）绝不自动重试。
