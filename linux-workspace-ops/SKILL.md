---
name: linux-workspace-ops
description: Eta Linux 环境使用与故障修复：workspace 与共享目录映射、包管理换源、DNS/代理故障修复、daemon 长驻服务、Android 与 Linux 文件搬运。装包失败、网络报错、跑脚本或长驻服务、批处理文件时使用。
---

# Linux Workspace Ops

Eta 的 Linux 环境（chroot/proot）操作手册。定位：跑脚本、装工具、批处理文件、长驻服务的工作间。

## 环境认知

- 命令用 `terminal` 工具，`environment=linux`（用户选定的发行版）；Android 侧用 `environment=android`，不要混用。
- **路径映射**：`/workspace` 映射宿主工作区；用户共享的手机目录挂在 `/workspace/mounts/<名>`——提到手机文件先 `ls /workspace/mounts/` 确认，再读写对应子目录。
- `read_image` 可直接读 `/workspace` 路径，不需要先拷到 Android 侧。
- 返回 `LINUX_ENVIRONMENT_NOT_READY` → 告知用户到设置里安装 Linux 环境；**不要**把 Linux 缺命令误报成设备不支持。
- 会长会话用 `action=open` 拿 session_id 后复用，不要每条命令新开会话。

## 网络故障修复（最高频）

症状与修法（详细排查树见 `references/network-troubleshooting.md`）：

| 症状 | 根因 | 修法 |
|------|------|------|
| npm 报 `EAI_AGAIN` / `getaddrinfo` 失败 | 域名解析不通（Android 拦明文 DNS，代理没接管 chroot） | 让 shell 走代理：`source /etc/profile.d/99eta-proxy.sh`（存在时）；不存在则探测代理端口写入 |
| apt 报 exit 70 / 索引为空 | 同上，apt 没走代理 | 写 `/etc/apt/apt.conf.d/99eta-proxy` 指向代理端口 |
| npm 下载慢/装不齐 | 默认官方源墙内不可达，optionalDependencies 被静默跳过 | **换镜像源**：`--registry=https://registry.npmmirror.com`，且重试也留在镜像源上 |
| 明明昨天能装今天失败 | 手机代理状态变了（开关/端口） | 重新探测代理端口，更新配置 |

**代理端口探测**（必须用 `bash -c`，`/dev/tcp` 是 bash 特性，sh/dash 不支持）：

```bash
for p in 7080 7890 7897 1080 10808; do
  bash -c "[ -d /proc ] && (echo >/dev/tcp/127.0.0.1/$p) 2>/dev/null" && echo "proxy=$p" && break
done
```

命中后 `export http_proxy=http://127.0.0.1:$p https_proxy=... all_proxy=...`。

## 包安装纪律

1. **装完必验**：`<tool> --version` 退出码 0 才算装上；报错原文保留给用户看。
2. npm 全局装：`npm install -g --registry=https://registry.npmmirror.com <pkg>`；失败别乱换源重试（官方源墙内必挂），先读报错。
3. apt 换源用国内镜像（清华/阿里）；Ubuntu arm64 用 **ubuntu-ports** 而不是 ubuntu。
4. 脚本里 source 代理文件用 `if [ -r /path ]; then . /path; fi`——`&&` 链在 `set -e` 下文件缺失会中断整个脚本。

## 长驻服务用 daemon

监听端口、Web 面板等长期进程：`daemon_start` 启动，`daemon_list` 看状态，`daemon_logs` 读日志，`daemon_stop` 停止。**不要用 nohup/& 自己后台化**。服务就绪前先探测端口/读日志确认，不盲等。

## 文件批处理

- 批量操作先 `ls` 确认清单 → 小批量执行 → 每批验证 → 出错即停。
- 删除类操作：先列出将被删除的完整清单给用户确认；Linux 侧没有回收站。
- 处理手机上的文件：经 `/workspace/mounts` 进出，不在 mount 里放中间产物（占手机存储）。

## 边界

- 不碰设备上其他终端 App（Termux 等）的环境和文件。
- 不卸载已装工具链；只加不减。
- 故障信息如实报告：命令、退出码、输出尾部，不吞错。
