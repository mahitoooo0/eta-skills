# 网络故障排查树

按现象入口往下查，命中即修，修完重试验证。

## 入口 1：npm 装包失败

```
npm install 报错
 ├─ EAI_AGAIN / getaddrinfo → DNS 解析不通 → 走「DNS/代理修复」
 ├─ ETIMEDOUT / ECONNREFUSED → 直连官方源不通 → 换镜像源
 │    npm install -g --registry=https://registry.npmmirror.com <pkg>
 │    （重试也必须留在镜像源；官方源墙内不可达）
 ├─ 装完但 CLI 跑不起来 / 二进制缺失
 │    → optionalDependencies 的平台二进制包被跳过
 │    → 原因通常是没走镜像或代理断了 → 修网络后重装
 │    → 兜底：apt install cmake build-essential（或 build-base）后源码编译重试（仍在镜像源上）
 └─ EACCES / permission → 检查安装前缀：npm -g 需要 --prefix /usr/local 或 root
```

验证：`<tool> --version` 退出码 0。

## 入口 2：apt 装包失败

```
apt update / install 报错
 ├─ exit 70 / 忽略所有索引 → 代理没配 → 写 /etc/apt/apt.conf.d/99eta-proxy：
 │    printf 'Acquire::http::Proxy "http://127.0.0.1:7080";\nAcquire::https::Proxy "http://127.0.0.1:7080";\n' > /etc/apt/apt.conf.d/99eta-proxy
 │    （端口按探测结果；键名必须写全 Acquire，缩写会被 apt 忽略）之后 apt update 验证
 ├─ 404 / 无发行版索引 → 源配置错 → arm64 Ubuntu 用 ubuntu-ports 仓库（不是 ubuntu）
 ├─ DEB822 格式 → Ubuntu 24.04+ 源在 /etc/apt/sources.list.d/ubuntu.sources，
 │    改源要同时清理旧 .sources 文件，残留官方源必挂
 └─ 磁盘满 → df -h 检查
```

## DNS/代理修复（核心机制）

**为什么**：Android 拦截应用的明文 DNS；手机的代理（VPN/eBPF 模式）只接管宿主流量，chroot 的流量既解析不了域名又没进代理——两头堵。

**修法分三层**：

1. 会话级（最快验证）：`export http_proxy=... https_proxy=... all_proxy=...` 后重试。
2. 登录 shell 级：写 `/etc/profile.d/99eta-proxy.sh`（存在就直接 source）。
3. 工具级：apt 写 `apt.conf.d/99eta-proxy`；npm/pnpm 写 `/root/.npmrc` 或 `/usr/local/etc/npmrc` 加 `registry=https://registry.npmmirror.com` 和 `proxy/https-proxy`。

**探测端口**（bash -c，原因：/dev/tcp 是 bash 内建；系统 sh 指向 dash）：

```bash
for p in 7080 7890 7897 1080 10808; do
  if bash -c "(echo >/dev/tcp/127.0.0.1/$p) 2>/dev/null"; then
    echo "PROXY_PORT=$p"; break
  fi
done
```

常见端口对应：7080=boxproxy(sing-box mixed)、7890=Clash、7897=Clash Verge、1080=通用 SOCKS、10808=v2rayN。

## 入口 3：服务连不上（127.0.0.1 端口拒绝）

```
浏览器/客户端访问 127.0.0.1:<port> 报连接拒绝
 ├─ daemon_list 看服务还活着吗；不活 → 看 daemon_logs 找崩溃原因
 ├─ 服务有认证（URL 带 token）→ 从启动日志取完整带认证地址，不要只开裸端口
 ├─ 只监听 IPv6 (::1) → 用 [::1]:port 或改服务配置绑 0.0.0.0 不被允许时用 127.0.0.1
 └─ chroot 与宿主共享网络命名空间 → localhost 理论可达；不通先确认服务进程存在（ps aux | grep）
```

## 入口 4：git clone 卡住

```
clone 长时间无输出
 ├─ 手机代理开着：确认 git 走了代理（git config --global http.proxy）
 ├─ 大仓库：git clone --depth 1；仍慢换 --filter=blob:none
 └─ 网络不行 → 放弃 clone，改 GitHub API 逐文件（见 coding-repo-workflow）
```

## 修复后的统一验证

1. `curl -sI https://registry.npmmirror.com --max-time 10` → HTTP 200。
2. 重跑原命令，保留退出码与输出尾部。
3. 诊断结论落进回复：改了哪个文件、为什么、怎么回滚（删掉写入的配置即可）。
