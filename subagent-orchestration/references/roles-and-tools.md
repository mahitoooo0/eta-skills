# 子代理角色与工具白名单

## 子代理能用的工具（只读白名单）

子代理没有 shell、没有 Android GUI、不能再委派、不能写主工作区。可用工具分三类：

**环境与设备状态（只读）**：get_current_context、search_apps、device_status、network_info、top_memory_apps、top_storage_apps、get_setting、get_current_location、get_device_environment、list_alarms、list_active_timers

**个人数据检索（只读）**：recent_notifications、search_notification_history、recent_app_activity、app_usage_summary、get_health_summary、search_media、search_audio、search_recordings、search_files、search_calendar_events、search_contacts、search_call_history、search_messages、search_downloads、search_personal_orders、search_qq_chat_images、search_wechat_chat_images

**文件与知识**：read_file、list_directory、skills_list、skills_read、skills_read_resource、memory_get

例外：
- **implementation 角色**不用白名单，改用 workspace_file 系列工具，只能写分配给它的 worktree。
- 文本子代理可带独立浏览器（browser_access=full/read_only/disabled），授权在任务创建时冻结，继续任务不会升级。

## 超时规则

| 任务类型 | 超时 | 性质 |
|---------|------|------|
| 文本（research/review/summary/implementation） | 360s | 软警告，有进展继续；无进展在安全点暂停 |
| 文本上下文压缩 | 360s | 终止 |
| 图片生成 | 180s | 终止，不自动重发（计费） |
| 视频生成 | 600s | 终止，不自动重发（计费） |

## get_task_result 事件流

- 不带 task_id → 列出本会话任务（最新在前，20/页）。
- `wait_ms`（≤10000）阻塞等事件；`after_seq`/`event_limit` 分页回读白名单事件。
- checkpoint 只是子代理自报的高层摘要；心跳 ≠ 进展；`oldest_seq`/`truncated` 表示日志页被覆盖。
- `can_replace`/`replace_reason` 只是建议，failed/paused 的任务永远不能当 completed 汇报。

## 实现代理的 Git worktree 语义

- 前置条件：`project` 指向 `/workspace` 下 Git 仓库根，且**无未提交变更**（有脏变更先让主代理处理）。
- 完成门槛：运行时验证过的**非空 Git 产物**；空 diff 或全部回滚 → `NO_IMPLEMENTATION_CHANGES` 失败。
- 构建和测试由主代理执行，实现代理只写代码。
- 合并只能 fast-forward（`manage_agent_workspace action=merge`）；合并前应有 review 通过 + 主代理亲自核验 diff；`WORKSPACE_IN_USE` 表示还有子任务占用，先等或取消。
- 没有自动 push：合并后的 push 由主代理按 coding-repo-workflow 的回写规则执行。

## 给子代理写任务书的模板

```
目标：<一句话结果>
范围：只动 <文件/目录>；不碰 <明确排除>
输入：<必要上下文、路径、参数，自包含>
验收：<可检查的标准>
报告：完成后列出改动文件与验证证据；不确定的列为假设
```
