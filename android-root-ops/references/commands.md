# Root Shell 命令速查

环境：Android，root 身份（`terminal` 工具 environment=android，root 可用时）。

## 包管理

```bash
pm list packages -f                    # 包名 + apk 路径
pm list packages | grep <kw>
pm path <pkg>                          # apk 位置
pm dump <pkg> | grep -E 'version|permission' | head
am force-stop <pkg>                    # 杀进程（写操作，先确认）
```

## 诊断

```bash
dumpsys battery
dumpsys batteryinfo | head -80
dumpsys meminfo
dumpsys cpuinfo | head -30
dumpsys activity top
dumpsys window | grep mCurrentFocus    # 当前前台界面
dumpsys deviceidle | head -30          # doze 状态
top -n 1 -m 10                        # 前 10 进程
ps -A | grep <pkg>
```

## 日志

```bash
logcat -d -t 500                       # 最近 500 行
logcat -d *:E -t 300                   # 只看 error
logcat -d --pid=$(pidof <pkg>) -t 200
logcat -d -s ActivityManager:E        # 按标签过滤
dmesg | tail -50                       # 内核日志
```

## 存储

```bash
df -h
du -h -d 1 /data 2>/dev/null | sort -rh | head -15
du -h -d 2 /data/data/<pkg> | sort -rh | head
ls -la /sdcard/Android/data/<pkg>/    # 应用外部私有目录
```

## 数据库与配置

```bash
sqlite3 <db> ".tables"
sqlite3 <db> ".schema <table>"
sqlite3 <db> "SELECT COUNT(*) FROM <t>;"
sqlite3 <db> "SELECT * FROM <t> LIMIT 20;"
cat /data/data/<pkg>/shared_prefs/*.xml | head -100
```

无 sqlite3 时：`cp <db> /workspace/` 交 Linux 环境处理。

## 设置与网络

```bash
settings get global http_proxy
settings list global | grep -i proxy
ifconfig | grep -A1 wlan
ip route
netstat -tlnp 2>/dev/null | head -20
ss -tlnp 2>/dev/null | head -20
```

## 小心清单（读也谨慎）

- `/data/misc/`、`/data/system/`：系统核心数据，只 cat 查看，绝不写。
- `settings put`、`pm clear`、`pm uninstall`、`rm`、`am start` 带敏感参数：全部属于写/变更操作，先确认。
- logcat 里出现 token、密码、验证码：不回显原文。
