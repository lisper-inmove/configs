# netproxy — 让不能上网的机器借用 A 的网络

## 拓扑

```
A（另一台 VMware 虚拟机，能上网，跑着 clash :7897）
   │  A 能 SSH 到 B；B 绝对不能主动连 A
   │  所以隧道必须由 A 发起（ssh -R），不是 B 发起
   ▼
B（本机，10.100.129.13，VMware 虚拟机，无公网、无可用 DNS）
   127.0.0.1:17891  ←── ssh 隧道 ──→  A 的 127.0.0.1:7897
   │
   │  relay（链式接力，可选）
   ▼
C（10.100.129.21，同样无公网）
   127.0.0.1:17892  ←── ssh -R（由 B 发起）──→  B 的 127.0.0.1:17891
```

因为 **B 无法主动连 A**，所以：

- **A 是发起方**，主隧道的开关在 A 上（`netproxy up` / `netproxy down`）
- **B 是被动方**，用 `netproxy status` / `env` / `run` / `test`
- **C 通过 B 中继**，B 用 `relay` 角色主动连 C（B→C 方向可达）

## 角色一览

| 角色 | 跑在哪 | 做什么 | 开关 |
|---|---|---|---|
| `a` | A | `ssh -R` 把 A 的 clash 端口推到 B | `netproxy up/down`（在 A 上） |
| `b` | B / C | 被动使用 `127.0.0.1:<port>` | 由上游控制 |
| `relay` | B | 把 B 已有的代理端口再接力给 C | `netproxy up/down`（在 B 上，独立配置文件） |

## 两种模式

| 模式 | ssh 参数 | 出口 |
|---|---|---|
| `forward`（默认，推荐） | `-R 127.0.0.1:17891:127.0.0.1:7897` | 走 **A 的 clash**，能访问 Google 等 |
| `dynamic` | `-R 127.0.0.1:17891` | sshd 直拨，走 A 的**原始线路**，被墙站点不通 |

⚠️ **这两个模式的行为差异极大，务必确认生效的是哪个。** 早期版本只有 dynamic，
所以出现过"baidu 通 / google 不通"的现象 —— 那不是隧道坏了，是没经过 clash。

相关配置：

```bash
NETPROXY_MODE=forward           # forward | dynamic
NETPROXY_UPSTREAM_PORT=7897     # A 上 clash 的端口（forward 模式）
NETPROXY_PROXY_SCHEME=socks5h   # socks5h | http
```

隧道只是搬运 TCP 端口，**A 上 clash 的协议必须和 B 用的 scheme 一致**。
`netproxy test` 会自动探测：

```
== 上游协议探测
  SOCKS5 : 可用
  HTTP   : 可用
  建议   : NETPROXY_PROXY_SCHEME=socks5h
```

## 部署

脚本只有一个文件 `netproxy`，两台各放一份，**版本必须一致**（`netproxy version`）。

需要 `~/.bin` 在 PATH 里（B 的 `~/.zshrc` 已有；A 视情况自行添加）。

### A 侧（发起方）

```bash
install -Dm755 netproxy ~/.bin/netproxy

mkdir -p ~/.config/netproxy
cat > ~/.config/netproxy/config <<'EOF'
NETPROXY_ROLE=a
NETPROXY_PEER=10.100.129.13
NETPROXY_PEER_USER=JC1136
NETPROXY_PEER_PORT=22
NETPROXY_IDENTITY=/home/JC1136/.ssh/id_rsa
NETPROXY_LOCAL_PORT=17891
NETPROXY_SOCKS_BIND=127.0.0.1
NETPROXY_MODE=forward
NETPROXY_UPSTREAM_PORT=7897
NETPROXY_PROXY_SCHEME=socks5h
EOF

ssh -i ~/.ssh/id_rsa JC1136@10.100.129.13 hostname   # 确认能登录 B
netproxy version                                     # 确认是 2.0
netproxy up
```

成功的输出应该是：

```
已开启：B 的 127.0.0.1:17891  ->  A 的 127.0.0.1:7897（经 JC1136@10.100.129.13）
代理地址：socks5h://127.0.0.1:17891
```

如果看到的是"现在是 SOCKS5 代理"，说明跑的是旧版脚本（dynamic 模式）。

### B 侧（本机）—— 已部署

```bash
~/.config/netproxy/config:
NETPROXY_ROLE=b
NETPROXY_PEER=
NETPROXY_PEER_USER=JC1136
NETPROXY_PEER_PORT=22
NETPROXY_IDENTITY=
NETPROXY_LOCAL_PORT=17891
NETPROXY_SOCKS_BIND=127.0.0.1
NETPROXY_MODE=forward
NETPROXY_UPSTREAM_PORT=7897
NETPROXY_PROXY_SCHEME=socks5h
```

从 A 拉取最新脚本：

```bash
# 在 A 上执行
scp JC1136@10.100.129.13:/home/JC1136/code/network/netproxy ~/.bin/netproxy
chmod 755 ~/.bin/netproxy && netproxy version
```

## B 侧用法

```bash
netproxy status                            # 端口状态 + 当前模式
netproxy test                              # 协议探测 + 实际出网测试
netproxy version                           # 版本/模式/端口

eval "$(netproxy env)"                     # 当前 shell 走代理
netproxy run curl -I https://github.com     # 只让一条命令走代理
netproxy shellinit >> ~/.zshrc             # 新 shell 自动生效（代理没起来时什么都不做）
```

退出代理环境：

```bash
unset http_proxy https_proxy all_proxy HTTP_PROXY HTTPS_PROXY ALL_PROXY
```

**必须用 `socks5h://`（末尾的 h）** —— B 的 DNS 完全不可用，要靠 A 侧解析域名。

## 让 SSH / git 也走隧道

`http_proxy` 只影响 HTTP(S) 客户端，**对 SSH 无效**。B 又解析不了 `ssh.github.com`，
所以 `git clone git@github.com:...` / `git push` 会直接报：

```
ssh: Could not resolve hostname ssh.github.com: Temporary failure in name resolution
```

解决办法是给 SSH 配一个 ProxyCommand。这里用一个小包装脚本 `~/.ssh/github-proxy`，
**隧道开着走 SOCKS5，没开则退回直连**（避免忘记开隧道时 SSH 卡死）：

```bash
#!/usr/bin/env bash
set -u
host="${1:?host}"; port="${2:?port}"
socks="127.0.0.1:${NETPROXY_LOCAL_PORT:-17891}"
if (exec 3<>"/dev/tcp/${socks%:*}/${socks#*:}") 2>/dev/null; then
  exec nc -X 5 -x "$socks" "$host" "$port"
fi
exec nc "$host" "$port"
```

然后写进 `~/.ssh/config`（这个文件已入库在本仓库中）：

```
Host github.com
  Hostname ssh.github.com
  Port 443
  ProxyCommand /home/JC1136/.ssh/github-proxy %h %p
```

验证：

```bash
ssh -G github.com | grep proxycommand     # 确认配置生效
ssh -T git@github.com                     # Hi lisper-inmove! ...
git ls-remote origin                      # 能列出分支即可
```

之后 `git push` / `git pull` 不需要再带任何额外参数。

> `nc` 必须是 OpenBSD netcat（支持 `-X 5 -x`）。若换成 nmap-ncat，
> 参数要改成 `--proxy-type socks5 --proxy 127.0.0.1:17891`。

## 第三台机器：链式中继（relay）

如果还有一台 **C（10.100.129.21）** 也需要借 A 的网，而 **C 也不能主动连 A**、
但 **B 能主动连 C**，就在 B 上做一次接力：

```
A ──ssh -R──► B:17891 ──ssh -R(relay)──► C:17892
```

### B 侧配置（独立文件，不动主配置）

```bash
# ~/.config/netproxy/relay.conf
NETPROXY_ROLE=relay
NETPROXY_PEER=10.100.129.21        # C 的地址
NETPROXY_PEER_USER=JC1136
NETPROXY_PEER_PORT=22
NETPROXY_LOCAL_PORT=17892          # C 上要监听的端口
NETPROXY_UPSTREAM_LOCAL_PORT=17891 # B 上已有的代理端口（由 A 的主隧道提供）
```

用独立配置文件，避免和主隧道共享 pid/日志：

```bash
NETPROXY_CONFIG=~/.config/netproxy/relay.conf netproxy up      # 开启
NETPROXY_CONFIG=~/.config/netproxy/relay.conf netproxy status
NETPROXY_CONFIG=~/.config/netproxy/relay.conf netproxy down    # 关闭
```

`up` 会先检查 B 上的 `17891` 是否存在，没有就拒绝启动并提示先开主隧道。
relay 的 pid/日志带 `-relay` 后缀（`ssh-relay.pid` / `netproxy-relay.log`），
与主隧道互不影响。

前置条件：**B 必须能免密 SSH 到 C**（`BatchMode=yes`，不能交互输密码）。

### C 侧

```bash
install -Dm755 netproxy ~/.bin/netproxy
# ~/.config/netproxy/config
NETPROXY_ROLE=b
NETPROXY_LOCAL_PORT=17892
NETPROXY_PROXY_SCHEME=socks5h

netproxy status      # 应显示运行中
netproxy run curl -I https://www.google.com.hk
```

C 上的 `~/.ssh/config` 也要配 ProxyCommand（端口改成 17892），否则 `git`
走 SSH 时同样会因 DNS/网络不通而失败：

```
Host github.com
  Hostname ssh.github.com
  Port 443
  ProxyCommand /home/JC1136/.ssh/github-proxy %h %p
```

### relay 已验证

| 检查 | 结果 |
|---|---|
| C 的 `127.0.0.1:17892` 监听 | ✅ |
| google.com.hk / github / pypi 经 C 访问 | 200 / 200 / 200 |
| 出口 IP | 美国 fdcservers（与 A 的 clash 一致） |
| 幂等 up、`down` 后 C 端口清理 | ✅ |
| 杀掉 ssh 子进程后自动重连（C 端口 2 秒内恢复） | ✅ |
| 上游 17891 不存在时拒绝启动 | ✅ |
| C 上 `netproxy run` / `env` / `test` | ✅ |
| C 上 `ssh -T git@github.com` | ✅ `Hi lisper-inmove!` |

## 实测结果（2026-11-20，forward 模式）

```
google.com           302      google.com.hk   200     youtube.com     200
wikipedia.org        200      github.com      200     baidu.com       200
pypi.org             200      npmjs.org       200     api.anthropic   401(正常)
```

| 项目 | 结果 |
|---|---|
| 出口 IP | `2001:49f0:d077:8::5`（美国 fdcservers.net，即 A 的 clash 出口） |
| 稳定性 | google.com.hk 连续 20 次全部 200 |
| 直连对照 | baidu/google 直连均 000 |
| `git clone` / `netproxy run` / `eval env` | ✅ |
| `git push` / `git ls-remote`（SSH remote） | ✅ 通过 ProxyCommand 走隧道 |

`zh.wikipedia.org` 返回 301 是维基自己的语言重定向，不是故障；
`api.anthropic.com` 的 401 是缺 API key（请求确实到达了服务端）。

## 行为细节

- **断线自愈**：A 上的监督进程在 ssh 退出后按 2s→60s 退避重连
- **幂等**：重复 `up` 不会起第二份；`down` 可重复执行
- **失败可见**：连不上 B 时约 7 秒内失败退出（非 0），清理进程与监听端口
- **角色自检**：在 B 上执行 `netproxy up` 会被拦住并说明原因
- **只绑回环**：端点固定在 `127.0.0.1`，不监听局域网
- **不碰系统**：不改 sshd、不加 systemd 单元、不改路由防火墙、不需要 root

## 已知坑

**`no_proxy` 不支持 CIDR。** curl 7.68（B 上的版本）会忽略 `10.0.0.0/8` 这类写法，
导致内网流量被送进代理。默认值因此列的是具体地址：

```
localhost,127.0.0.1,::1,.local,10.100.129.13,10.100.129.0
```

如果你还要访问别的内网服务，用 `NETPROXY_NO_PROXY` 把它们逐个加上。

**git 的 URL 改写会绕过代理。** 如果 `~/.gitconfig` 里有这类配置：

```
url.https://ghfast.top/https://github.com/.insteadof = https://github.com/
```

github 流量会被改写到镜像站，而镜像站不在代理覆盖范围内，表现为
`gnutls_handshake() failed`。检查并删除：

```bash
git config --global --get-regexp 'url\.'          # 先确认有没有
git config --global --unset url.https://ghfast.top/https://github.com/.insteadOf
```

（B 当前已无此配置。）

**SSH 不读 `http_proxy`。** 详见上文「让 SSH / git 也走隧道」。

**环境变量优先于配置文件。** `NETPROXY_*=...` 写在命令前面会覆盖配置文件里的同名项，
临时改端口不用编辑文件：

```bash
NETPROXY_LOCAL_PORT=17893 netproxy up
```

**relay 模式用独立配置文件。** 同一个 `~/.config/netproxy/*.conf` 只能有一种角色，
所以中继要单独一份并通过 `NETPROXY_CONFIG` 指向它。
