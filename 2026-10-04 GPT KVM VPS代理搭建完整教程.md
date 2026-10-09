
# DediRock 洛杉矶 VPS + Hysteria 2 完整部署与优化指南

## 一、最终架构

```
Windows / Android / iPhone
          │
          │ Hysteria 2
          │ UDP / QUIC
          ▼
   hy.example.com:443
          │
          │ DNS
          ▼
 DediRock Los Angeles VPS
          │
     Ubuntu 24.04 LTS
          │
      Hysteria 2
          │
          ▼
       Internet
```

Cloudflare 只负责：

```
hy.example.com → VPS IP
```

使用 **DNS Only（灰云）**，不代理 Hysteria 2 的 UDP 流量。

---

# 二、购买 VPS

## 1. 推荐配置

```
地区：Los Angeles
CPU：1 vCPU
内存：2 GB
磁盘：30 GB SSD
流量：2 TB
端口：1 Gbps
IPv4：1 个
系统：Ubuntu 24.04 LTS
```

购买完成后记下：

```
VPS IP
SSH 用户名
SSH 密码
```

例如：

```
IP：1.2.3.4
用户名：root
```

不要公开 VPS 密码和 SSH 私钥。

---

# 三、准备域名

准备一个自己的域名。

假设：

```
example.com
```

给 Hysteria 建一个子域名：

```
hy.example.com
```

最终：

```
hy.example.com
       ↓
   VPS IP
```

---

# 四、配置 Cloudflare DNS

进入：

```
Cloudflare
→ 你的域名
→ DNS
→ Records
```

添加：

```
Type：A
Name：hy
IPv4：你的 VPS IP
Proxy status：DNS Only
TTL：Auto
```

必须使用：

```
☁ 灰色云朵
DNS Only
```

不要使用：

```
🟠 橙色云朵
Proxied
```

普通 Cloudflare Proxy 面向 HTTP/HTTPS；Hysteria 2 使用 UDP/QUIC，因此这里让 Cloudflare 只负责 DNS。([developers.cloudflare.com](https://developers.cloudflare.com/dns/proxy-status/?utm_source=chatgpt.com))

---

# 五、检查 DNS

Windows PowerShell：

```
nslookup hy.example.com
```

确认返回的 IP 是你的 VPS IP。

也可以：

```
ping hy.example.com
```

如果没有解析到 VPS，先解决 DNS 问题，再继续。

---

# 六、SSH 登录 VPS

Windows PowerShell：

```
ssh root@你的VPS_IP
```

例如：

```
ssh root@1.2.3.4
```

第一次连接：

```
Are you sure you want to continue connecting?
```

输入：

```
yes
```

然后输入 VPS 密码。

Linux 输入密码时通常不会显示字符，这是正常的。

---

# 七、更新 Ubuntu

```
apt update
apt upgrade -y
```

安装基础工具：

```
apt install -y curl wget nano openssl ufw htop
```

用途：

```
curl      下载/访问网络资源
wget      下载文件
nano      编辑配置文件
openssl   生成随机密码
ufw       防火墙
htop      查看 CPU / 内存
```

---

# 八、先测试 VPS 原始线路

先不要安装 Hysteria。

## 1. 测试公网连接

```
ping -c 10 1.1.1.1
```

```
ping -c 10 8.8.8.8
```

## 2. 查看 VPS 出口 IPv4

```
curl -4 ip.sb
```

应该显示你的 VPS IPv4。

这一阶段主要确认：

```
你的网络
   ↓
Internet
   ↓
DediRock LA
```

通信正常。

---

# 九、配置 VPS 防火墙

SSH：

```
ufw allow 22/tcp
```

Hysteria：

```
ufw allow 443/udp
```

启用：

```
ufw enable
```

查看：

```
ufw status
```

至少应该允许：

```
22/tcp
443/udp
```

## 如果 SSH 不是 22 端口

例如 SSH 使用 2222：

```
ufw allow 2222/tcp
```

确认以后再启用 UFW。

---

# 十、生成 Hysteria 2 密码

执行：

```
openssl rand -hex 24
```

例如：

```
9c4b6f0f2a3e7d1c8b5a0f4d6e9c2a7b1f8d3e5c6a0b4d9f
```

保存这个密码。

服务端和客户端必须使用同一个密码。

---

# 十一、安装 Hysteria 2

使用 Hysteria 官方安装脚本：

```
bash <(curl -fsSL https://get.hy2.sh/)
```

安装完成：

```
hysteria version
```

确认可以显示版本号。

官方说明，该脚本负责安装、升级和 systemd 服务管理，但不会自动完成完整的服务端配置。([v2.hysteria.network](https://v2.hysteria.network/zh/docs/getting-started/Server-Installation-Script/?utm_source=chatgpt.com))

---

# 十二、创建服务端配置

打开：

```
nano /etc/hysteria/config.yaml
```

写入：

```
listen: :443

acme:
  domains:
    - hy.example.com
  email: your@email.com

auth:
  type: password
  password: "你的随机密码"
```

替换：

```
hy.example.com
```

为你的域名。

替换：

```
your@email.com
```

为你的邮箱。

替换：

```
你的随机密码
```

为刚刚生成的密码。

---

# 十三、配置项说明

## `listen`

```
listen: :443
```

表示 Hysteria 监听 443 端口。

如果只需要 IPv4：

```
listen: 0.0.0.0:443
```

---

## `acme`

```
acme:
```

启用自动 TLS 证书申请。

---

## `domains`

```
domains:
  - hy.example.com
```

为这个域名申请证书。

---

## `email`

```
email: your@email.com
```

用于 ACME 证书申请。

---

## `auth`

```
auth:
  type: password
  password: "你的随机密码"
```

使用密码对客户端进行认证。

Hysteria 官方当前支持这种 ACME + Password 配置。([v2.hysteria.network](https://v2.hysteria.network/zh/docs/getting-started/Server/?utm_source=chatgpt.com))

---

# 十四、第一版不要添加高级参数

暂时不要加入：

```
QUIC Window
MTU
Brutal
BBR Profile
Obfs
Port Hopping
特殊 DNS
```

Hysteria 当前默认 QUIC 接收窗口已经有合理设置；官方明确建议不了解这些参数时不要随意修改。([v2.hysteria.network](https://v2.hysteria.network/docs/advanced/Full-Server-Config/?utm_source=chatgpt.com))

---

# 十五、启动 Hysteria

```
systemctl enable --now hysteria-server.service
```

查看：

```
systemctl status hysteria-server.service
```

正常：

```
Active: active (running)
```

其中：

```
enable
→ 开机自动启动

--now
→ 现在立即启动
```

---

# 十六、检查 UDP 443

```
ss -lunp | grep 443
```

再：

```
ufw status
```

确认：

```
443/udp
```

已经允许。

---

# 十七、检查 Hysteria 日志

如果启动失败：

```
journalctl --no-pager -e -u hysteria-server.service
```

实时查看：

```
journalctl -f -u hysteria-server.service
```

重点检查：

```
ACME
certificate
config
address already in use
permission denied
```

---

# 十八、检查 TLS / ACME

如果证书申请失败，按顺序检查：

```
① DNS 是否指向 VPS
② Cloudflare 是否 DNS Only
③ 域名是否写对
④ UDP/TCP 端口是否冲突
⑤ Hysteria 服务是否运行
⑥ 日志中的 ACME 错误
```

DNS：

```
nslookup hy.example.com
```

端口：

```
ss -lunp | grep 443
```

服务：

```
systemctl status hysteria-server.service
```

---

# 十九、Windows 配置 Clash Verge Rev

使用支持 Mihomo/Clash Meta 的 Clash Verge Rev。

在：

```
proxies:
```

中加入：

```
- name: "DediRock-LA-HY2"
  type: hysteria2
  server: hy.example.com
  port: 443
  password: "你的随机密码"
  sni: hy.example.com
  skip-cert-verify: false
```

Mihomo 当前支持 Hysteria 2 及这些基本参数。([wiki.metacubex.one](https://wiki.metacubex.one/en/config/proxies/hysteria2/?utm_source=chatgpt.com))

---

# 二十、Clash 参数说明

```
name:
```

Clash 中显示的节点名称。

```
type: hysteria2
```

使用 Hysteria 2。

```
server:
```

服务器域名。

```
port:
```

服务器端口。

```
password:
```

Hysteria 服务端密码。

```
sni:
```

TLS 使用的域名。

```
skip-cert-verify: false
```

正常验证 TLS 证书。

不要为了绕过证书问题设置：

```
skip-cert-verify: true
```

---

# 二十一、第一次连接测试

Clash Verge：

```
选择 DediRock-LA-HY2
↓
打开系统代理
```

然后检查出口 IP。

确认：

```
出口 IP
地区
ASN
```

对应 DediRock 洛杉矶 VPS。

---

# 二十二、测试实际服务

依次测试：

```
普通网站
GitHub
ChatGPT
Gemini
Claude
```

确认：

```
连接正常
DNS 正常
TLS 正常
不会频繁断线
```

---

# 二十三、第一次测速

使用：

```
https://speed.cloudflare.com/
```

记录：

```
延迟
下载
上传
```

建议记录：

```
时间
节点
Ping
丢包
下载
上传
```

例如：

```
14:00
DediRock-LA-HY2
155ms
0%
350Mbps
120Mbps
```

---

# 二十四、晚高峰测速

至少比较：

```
白天
晚上 8～10 点
```

例如：

```
白天
150ms
0%
350Mbps

晚上
190ms
2%
180Mbps
```

如果晚高峰明显变差，重点检查：

> 线路质量

而不是先修改 Hysteria 参数。

Hysteria 官方也将客户端与服务器之间的连接质量列为重要性能因素。([v2.hysteria.network](https://v2.hysteria.network/docs/advanced/Performance/?utm_source=chatgpt.com))

---

# 二十五、第一项性能优化：Linux UDP Buffer

基础连接稳定以后再优化。

执行：

```
sysctl -w net.core.rmem_max=16777216
sysctl -w net.core.wmem_max=16777216
```

表示：

```
最大接收缓冲区：16MB
最大接收发送区：16MB
```

这是 Hysteria 官方性能文档提供的 Linux 缓冲区优化设置。([v2.hysteria.network](https://v2.hysteria.network/docs/advanced/Performance/?utm_source=chatgpt.com))

---

# 二十六、永久保存 UDP Buffer

创建：

```
nano /etc/sysctl.d/99-hysteria.conf
```

写入：

```
net.core.rmem_max=16777216
net.core.wmem_max=16777216
```

应用：

```
sysctl --system
```

检查：

```
sysctl net.core.rmem_max
sysctl net.core.wmem_max
```

---

# 二十七、优化后重新测速

再次记录：

```
延迟
丢包
下载
上传
```

比较：

```
优化前
VS
优化后
```

如果没有明显改善，就不要继续盲目修改。

---

# 二十八、第二项性能优化：MTU

只有出现：

```
小网页正常
大文件异常
部分网站打不开
视频异常
UDP 异常
```

时，再检查 MTU。

Hysteria 默认提供 Path MTU Discovery，因此第一版不要主动关闭。([v2.hysteria.network](https://v2.hysteria.network/docs/advanced/Full-Server-Config/?utm_source=chatgpt.com))

---

# 二十九、第三项性能优化：QUIC 参数

Hysteria 包含：

```
Stream Receive Window
Connection Receive Window
```

当前默认值：

```
Stream：8MB
Connection：20MB
```

官方建议不了解这些参数时不要修改。([v2.hysteria.network](https://v2.hysteria.network/docs/advanced/Full-Server-Config/?utm_source=chatgpt.com))

因此：

```
第一版
→ 使用默认值

实测出现吞吐问题
→ 再针对性调整
```

---

# 三十、第四项性能优化：拥塞控制

Mihomo 当前支持 Hysteria 2 的：

```
up
down
bbr-profile
```

以及不同 BBR Profile。([wiki.metacubex.one](https://wiki.metacubex.one/config/proxies/hysteria2/?utm_source=chatgpt.com))

这些主要影响：

> 吞吐和拥塞控制

不是简单的：

> 开启后 Ping 一定降低。

所以不要默认加入。

---

# 三十一、第五项性能优化：DNS

DNS 的作用：

```
域名
↓
IP
```

后期可以比较：

```
Cloudflare DNS
1.1.1.1

Google DNS
8.8.8.8

其他 DNS
```

DNS 主要影响：

> 域名解析速度

不是直接降低 VPS 与你之间的 RTT。

---

# 三十二、第六项：Port Hopping

只有确认：

```
UDP 整体正常
但特定端口异常
```

时才考虑。

Hysteria 2 官方支持 Port Hopping，并说明它主要针对特定 UDP 端口受到限制的情况；如果整个 UDP 都被限制，端口跳跃没有帮助。([v2.hysteria.network](https://v2.hysteria.network/docs/advanced/Port-Hopping/?utm_source=chatgpt.com))

第一版不启用。

---

# 三十三、第七项：Obfuscation

Hysteria 2 支持流量混淆，例如：

```
Salamander
```

它主要用于：

> 网络对 QUIC/HTTP3 类流量存在阻断或识别问题

不是：

> 降低延迟。

正常使用时不需要打开。([v2.hysteria.network](https://v2.hysteria.network/docs/advanced/Full-Client-Config/?utm_source=chatgpt.com))

---

# 三十四、Cloudflare 在这套方案中的正确位置

最终保持：

```
Cloudflare
    │
    │ DNS Only
    ▼
hy.example.com
    │
    ▼
DediRock VPS
```

Cloudflare 不参与：

```
Hysteria UDP 数据传输
```

不要将 Hysteria 域名改成普通 Cloudflare：

```
🟠 Proxied
```

普通 Cloudflare CDN 不能直接作为 Hysteria 2 UDP 流量的普通 CDN 代理。需要 TCP/UDP 代理能力时，对应的是 Spectrum 等独立产品。([developers.cloudflare.com](https://developers.cloudflare.com/spectrum/?utm_source=chatgpt.com))

---

# 三十五、什么时候考虑中转

如果测试结果：

```
白天正常
晚上明显变差
```

或者：

```
延迟始终很高
丢包严重
```

优先怀疑线路。

这时才考虑：

```
你的设备
    ↓
香港 / 日本 / 新加坡
    ↓
DediRock LA
    ↓
Internet
```

即：

> 中转

中转主要解决：

> 线路问题

而不是：

> Hysteria 本身性能不足。

---

# 三十六、SSH 安全加固

代理稳定运行以后再做。

生成 SSH Key：

```
ssh-keygen -t ed25519
```

配置完成后测试：

```
ssh -i 你的私钥 root@你的VPS_IP
```

确认 SSH Key 登录成功以后，再考虑关闭 SSH 密码登录。

不要在没有验证 SSH Key 前直接关闭密码登录。

---

# 三十七、常用命令

## Hysteria

```
systemctl start hysteria-server.service
systemctl stop hysteria-server.service
systemctl restart hysteria-server.service
systemctl status hysteria-server.service
```

## 日志

```
journalctl --no-pager -e -u hysteria-server.service
```

```
journalctl -f -u hysteria-server.service
```

## UDP 443

```
ss -lunp | grep 443
```

## 防火墙

```
ufw status
```

## CPU / 内存

```
htop
```

## DNS

```
nslookup hy.example.com
```

---

# 三十八、修改配置后的固定流程

每次修改：

```
nano /etc/hysteria/config.yaml
```

保存：

```
systemctl restart hysteria-server.service
```

检查：

```
systemctl status hysteria-server.service
```

异常：

```
journalctl --no-pager -e -u hysteria-server.service
```

固定流程：

```
修改
 ↓
重启
 ↓
检查状态
 ↓
查看日志
```

---

# 三十九、完整最终配置

## Cloudflare

```
A
hy
VPS IP
DNS Only
```

## Hysteria 服务端

文件：

```
/etc/hysteria/config.yaml
```

内容：

```
listen: :443

acme:
  domains:
    - hy.example.com
  email: your@email.com

auth:
  type: password
  password: "你的强随机密码"
```

## Linux 网络优化

文件：

```
/etc/sysctl.d/99-hysteria.conf
```

内容：

```
net.core.rmem_max=16777216
net.core.wmem_max=16777216
```

## Clash

```
proxies:
  - name: "DediRock-LA-HY2"
    type: hysteria2
    server: hy.example.com
    port: 443
    password: "你的强随机密码"
    sni: hy.example.com
    skip-cert-verify: false
```

---

# 四十、最终推荐的优化策略

按照以下顺序进行，不要跳步：

```
VPS 线路
   ↓
UDP 质量
   ↓
Hysteria 2 默认配置
   ↓
Clash
   ↓
延迟 / 丢包 / 下载 / 上传测试
   ↓
Linux UDP Buffer
   ↓
重新测试
   ↓
MTU
   ↓
QUIC 参数
   ↓
拥塞控制
   ↓
Port Hopping / Obfs
   ↓
中转
```

每次只修改一个变量：

```
修改前测速
    ↓
修改一个参数
    ↓
重新测速
    ↓
比较结果
```

---

# 四十一、常见问题排查

## DNS 错误

```
Cloudflare
↓
A
↓
hy.example.com
↓
VPS IP
```

检查：

```
nslookup hy.example.com
```

---

## Hysteria 启动失败

```
systemctl status hysteria-server.service
```

然后：

```
journalctl --no-pager -e -u hysteria-server.service
```

---

## 连接超时

依次检查：

```
域名
↓
VPS IP
↓
443/UDP
↓
UFW
↓
Hysteria
↓
密码
↓
SNI
↓
TLS
```

---

## 证书错误

重点检查：

```
DNS 是否正确
Cloudflare 是否 DNS Only
域名是否写对
ACME 日志
```

---

## 速度慢

依次检查：

```
线路
↓
UDP
↓
丢包
↓
CPU
↓
UDP Buffer
↓
MTU
↓
QUIC
↓
拥塞控制
```

---

# 四十二、部署完成检查表

```
□ DediRock LA VPS
□ Ubuntu 24.04
□ 公网 IPv4
□ 域名
□ Cloudflare DNS
□ hy.example.com → VPS IP
□ Cloudflare DNS Only
□ SSH 正常
□ UFW 正常
□ 22/TCP
□ 443/UDP
□ Hysteria 2 已安装
□ config.yaml 正确
□ ACME 正常
□ TLS 正常
□ 强随机密码
□ systemd 正常
□ Hysteria 正常运行
□ Clash Verge Rev
□ Mihomo
□ Hysteria 2 节点
□ SNI 正确
□ TLS 验证开启
□ 出口 IP 正确
□ 延迟测试
□ 丢包测试
□ 下载测试
□ 上传测试
□ 晚高峰测试
□ UDP Buffer 优化
```

---

# 四十三、关键名词速记

|名词|作用|
|---|---|
|VPS|远程服务器|
|IP|VPS 的网络地址|
|SSH|远程登录服务器|
|Ubuntu|VPS 操作系统|
|Domain|域名|
|DNS|域名解析到 IP|
|Port|网络服务入口|
|UDP|Hysteria 2 使用的传输方式|
|QUIC|建立在 UDP 上的传输协议|
|Hysteria 2|代理服务|
|TLS|加密和身份验证|
|ACME|自动申请 TLS 证书|
|Let's Encrypt|免费 TLS 证书机构|
|Clash Verge|Windows 代理客户端|
|Mihomo|Clash 使用的核心|
|UFW|Ubuntu 防火墙|
|systemd|Linux 服务管理|
|Buffer|网络数据缓冲区|
|MTU|网络数据包最大传输单元|
|BBR|拥塞控制算法|
|Port Hopping|UDP 端口跳跃|
|Obfs|流量混淆|

---

# 四十四、完整实际操作顺序

```
1. 购买 DediRock LA VPS
        ↓
2. 选择 Ubuntu 24.04
        ↓
3. 获取 VPS IP / SSH
        ↓
4. 准备域名
        ↓
5. Cloudflare 添加 A 记录
        ↓
6. 设置 DNS Only 灰云
        ↓
7. nslookup 检查域名
        ↓
8. SSH 登录 VPS
        ↓
9. 更新 Ubuntu
        ↓
10. 安装基础工具
        ↓
11. 测试 VPS 原始线路
        ↓
12. 配置 UFW
        ↓
13. 开放 22/TCP
        ↓
14. 开放 443/UDP
        ↓
15. 安装 Hysteria 2
        ↓
16. 生成随机密码
        ↓
17. 编辑 config.yaml
        ↓
18. 配置 ACME
        ↓
19. 启动 Hysteria
        ↓
20. 检查 systemd
        ↓
21. 检查 UDP 443
        ↓
22. 检查 ACME/TLS
        ↓
23. 配置 Clash Verge
        ↓
24. 导入 Hysteria 2
        ↓
25. 开启系统代理
        ↓
26. 检查出口 IP
        ↓
27. 测试网站 / AI
        ↓
28. 测试延迟
        ↓
29. 测试丢包
        ↓
30. 测试下载/上传
        ↓
31. 测试晚高峰
        ↓
32. 优化 Linux UDP Buffer
        ↓
33. 再次测速
        ↓
34. 有具体问题再优化 MTU / QUIC
        ↓
35. 必要时调整拥塞控制
        ↓
36. 必要时启用 Port Hopping / Obfs
        ↓
37. 如果线路本身不好，再考虑中转
        ↓
38. 最后进行 SSH 安全加固
```

---

# 四十五、最终原则

第一版：

```
DediRock LA
+
Ubuntu 24.04
+
Cloudflare DNS Only
+
Hysteria 2
+
TLS / ACME
+
UDP 443
+
Clash Verge
```

先确保：

```
能连接
+
低丢包
+
速度正常
+
稳定
```

然后再优化。

不要一开始同时加入：

```
Cloudflare CDN
Nginx
WebSocket
QUIC Window 魔改
MTU 魔改
Brutal
BBR Profile
Obfs
Port Hopping
多级中转
```

每次只调整一个变量，通过实际测速判断是否有效。

核心指标：

```
低延迟
+
低丢包
+
高吞吐
+
稳定
```

而不是单纯追求某一个测速数字。