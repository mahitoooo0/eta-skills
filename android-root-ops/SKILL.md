---
name: android-root-ops
description: 用 root shell 只读探查 Android 系统：应用私有数据与数据库、系统诊断（dumpsys/logcat）、存储与电池分析。用户问应用数据在哪、为什么卡/耗电/占空间时使用。
---

# Android Root Ops

设备已 root 时的系统探查方法。**总纪律：默认只读**；任何写操作（改文件、clear data、卸载、改设置）先列对象、影响、风险，经用户确认。

## 路径选择

1. **优先专用只读工具**：通知、日历、联系人、通话、短信、媒体、文件都有专门检索工具，先用它们。
2. debug 版应用可用 `run-as <pkg>` 读私有目录，不需要 root。
3. 专用工具不够时，root shell 只读检查（root 不可用时直接说明，不试 su）。

## 探查应用私有数据

```bash
pm list packages | grep <关键词>          # 找包名
ls /data/data/<pkg>/                      # databases/ shared_prefs/ files/ cache/
run-as <pkg> ls databases                 # debug 包免 root
```

查数据库的正确姿势（**先 schema 后有界查询**）：

```bash
sqlite3 /data/data/<pkg>/databases/xxx.db ".tables"
sqlite3 /data/data/<pkg>/databases/xxx.db ".schema <table>"
sqlite3 /data/data/<pkg>/databases/xxx.db "SELECT * FROM <table> LIMIT 20;"
```

- 不 `SELECT *` 全表、不整库 dump；先看行数 `SELECT COUNT(*)`。
- 没有 sqlite3 二进制时，把 db 文件复制到 /workspace 再用 Linux 环境的工具处理（复制是读操作，安全）。
- shared_prefs 是 XML，直接读即可。
- 读到隐私内容（聊天记录、位置、凭据）只在回答里概括，不整段外传，不写入其他文件。

## 系统诊断

```bash
dumpsys battery                       # 电池/耗电异常
dumpsys meminfo                       # 内存压力
dumpsys activity top | head -50        # 当前前台应用
dumpsys package <pkg> | grep -A5 version  # 版本、权限
logcat -d -t 300 *:E                   # 最近错误日志
logcat -d --pid=$(pidof <pkg>) -t 200  # 按应用过滤
settings get global http_proxy         # 系统代理设置
du -h -d 2 /data/data/<pkg> 2>/dev/null | sort -rh | head  # 应用占用
df -h                                  # 磁盘空间
```

## 常见任务套路

| 任务 | 步骤 |
|------|------|
| 「什么占空间」 | df -h → du 逐层下钻大目录 → 报告前 10 与大小，只报告不删 |
| 「为什么这么卡」 | dumpsys meminfo + top_memory_apps → 前台应用 logcat 错误 → 结论区分观察与推测 |
| 「这应用把数据存哪」 | pm list 找包名 → ls 私有目录 → 有 db 先 .schema |
| 「耗电异常」 | dumpsys battery + battery_history 相关 → 定位 wakelock/后台进程 |

## 红线

- 不修改、不删除应用与系统数据；不 rm 任何 /data 路径。
- 不把隐私数据原文写进日志、文件或回复。
- 探查结果里区分「已核实」与「推测」，不编造没查到的数据。

命令速查表见 `references/commands.md`。
