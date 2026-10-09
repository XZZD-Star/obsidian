# DediRock VPS SSH 无法连接问题排查与解决交接文档

> 用于后续恢复上下文，以及以后再次遇到 FinalShell / PowerShell 无法连接 VPS 时快速排查。
> 
> 当前 VPS：
> 
> - 服务商：DediRock
>     
> - 地区：Los Angeles
>     
> - IPv4：`107.150.26.37`
>     
> - 系统：Ubuntu 24.04.4 LTS
>     
> - Hostname：`dedirock-104990068`
>     
> 
> 当前最终状态：
> 
> **VPS 正常，SSH 正常，问题最终定位到 Windows 本机 Clash Verge 虚拟网卡/TUN 网络状态异常。删除虚拟网卡后 SSH 立即恢复，重新安装虚拟网卡后也恢复正常。**

---

# 一、最初出现的问题

原本使用 FinalShell 连接 VPS：

```
107.150.26.37:22
root
```

突然出现：

```
connection is closed by foreign host
```

以及之前出现过：

```
SSH_MSG_DISCONNECT: 2 Too many authentication failures
```

后来 PowerShell 使用 OpenSSH 测试：

```
ssh -v root@107.150.26.37
```

最后出现：

```
kex_exchange_identification: Connection closed by remote host
Connection closed by 107.150.26.37 port 22
```

说明不是简单的“端口不通”，而是：

```
电脑
  ↓
107.150.26.37:22
  ↓
TCP 可以建立
  ↓
SSH 握手阶段异常
  ↓
连接被关闭
```

---

# 二、第一步：确认 VPS 是否在线

进入 DediRock 后台：

```
DediRock
→ Virtual Server
→ VPS
```

发现：

```
Status: Online
IP: 107.150.26.37
Region: Los Angeles
```

说明：

```
VPS 虚拟机本身没有关机
```

注意：

> DediRock 显示 Online 只能证明虚拟机处于运行状态，不能证明 Ubuntu 里面的 SSH、3x-ui、Xray 等服务全部正常。

---

# 三、第二步：排除 Windows Clash 对 SSH 的影响

最开始使用：

```
Test-NetConnection rcardoshaw.de5.net -Port 2096
```

出现：

```
InterfaceAlias : Meta
RemoteAddress : 198.18.0.205
SourceAddress : 198.18.0.1
TcpTestSucceeded : True
```

这里发现：

```
InterfaceAlias : Meta
```

说明 Clash / Mihomo 虚拟网络接口参与了流量处理。

同时出现：

```
198.18.x.x
```

属于 Clash/Mihomo 常见的 Fake-IP / 虚拟网络地址范围。

因此当时的测试不能直接证明：

```
电脑 → VPS
```

是正常直连的。

---

# 四、关闭 Clash 后重新测试 VPS 22 端口

关闭：

```
Clash Verge
TUN
系统代理
```

然后执行：

```
Test-NetConnection 107.150.26.37 -Port 22
```

结果：

```
ComputerName     : 107.150.26.37
RemoteAddress    : 107.150.26.37
RemotePort       : 22
InterfaceAlias   : WLAN
SourceAddress    : 192.168.10.201
TcpTestSucceeded : True
```

说明：

```
Windows
 ↓
正常 WLAN
 ↓
107.150.26.37:22
 ↓
TCP ✅
```

因此：

> VPS 的 22 端口可以从电脑建立 TCP 连接。

---

# 五、第三步：进入 DediRock Web Console 检查 Ubuntu

由于 FinalShell 一直连接不上，所以没有继续依赖 SSH。

通过 DediRock 后台：

```
VPS
→ 蓝色显示器图标
→ Web Console / KVM
```

直接进入 Ubuntu。

登录：

```
login: root
Password: VPS root 密码
```

成功进入：

```
root@dedirock-104990068:~#
```

说明：

```
VPS ✅
Ubuntu ✅
root 登录 ✅
```

---

# 六、第四步：检查 SSH 服务

执行：

```
systemctl status ssh
```

结果：

```
Active: active (running)
```

说明：

```
sshd 正常运行
```

同时看到：

```
Main PID: 35411 (sshd)
```

SSH 服务没有停止。

---

# 七、第五步：检查 22 端口监听

执行：

```
ss -lntp | grep ':22'
```

结果：

```
LISTEN 0 4096 0.0.0.0:22
LISTEN 0 4096 [::]:22
```

说明：

```
IPv4: 0.0.0.0:22 ✅
IPv6: [::]:22 ✅
```

SSH 正常监听 22 端口。

---

# 八、第六步：检查 SSH 配置语法

执行：

```
sshd -t
```

结果：

```
没有任何输出
```

这属于正常情况。

含义：

> `sshd_config` 配置语法没有错误。

---

# 九、第七步：重启 SSH 服务

执行：

```
systemctl restart ssh
```

然后：

```
systemctl status ssh --no-pager
```

确认：

```
Active: active (running)
```

重启日志显示：

```
Server listening on 0.0.0.0 port 22.
Server listening on :: port 22.
Started ssh.service - OpenBSD Secure Shell server.
```

说明 SSH 已重新正常启动。

但此时 Windows SSH 仍然：

```
Connection closed by 107.150.26.37 port 22
```

因此问题并没有因为重启 sshd 解决。

---

# 十、发现公网 SSH 扫描

SSH 日志中出现：

```
Invalid user admin from 2.57.121.25
Failed password for invalid user admin
```

说明 VPS 的 22 端口正在被公网自动扫描。

这是公网 VPS 很常见的情况。

这些日志不是自己的登录记录，也不是本次问题的直接证据。

---

# 十一、第八步：检查 MaxStartups 等 SSH 限制

执行：

```
sshd -T | grep -Ei 'maxstartups|persource|allowusers|denyusers|refuse'
```

结果：

```
maxstartups 10:30:100
persourcemaxstartups none
persourcenetblocksize 32:128
```

这些属于正常配置，没有发现：

```
AllowUsers
DenyUsers
RefuseConnection
```

等明显限制。

因此没有证据表明：

```
SSH 并发限制
SSH 来源限制
Allow/Deny 规则
```

是当前问题。

---

# 十二、第九步：检查 ssh.socket

第一次命令输入错误：

```
systemctl| status.ssh.socket --no-pager
```

导致：

```
command not found
```

后改为正确命令：

```
systemctl status ssh.socket --no-pager
```

结果：

```
ssh.socket - OpenBSD Secure Shell server socket

Active: active (running)

Listen:
0.0.0.0:22
[::]:22
```

说明：

```
ssh.socket ✅
监听 22 ✅
```

因此 socket activation 也正常。

---

# 十三、第十步：检查 Fail2ban

执行：

```
fail2ban-client status sshd
```

结果：

```
Currently failed: 0
Total failed: 3317

Currently banned: 0
Total banned: 597
```

解释：

```
Fail2ban 已安装 ✅
历史上封禁过 597 个 IP
当前封禁 IP：0
```

因此：

> 当前没有发现本机 IP 正被 Fail2ban 封禁。

不能把这次 SSH 问题直接归因于 Fail2ban。

---

# 十四、第十一步：检查 UFW

执行：

```
ufw status verbose
```

结果：

```
Command 'ufw' not found
```

说明：

```
UFW 没有安装
```

因此这台 VPS 当前没有通过 UFW 管理防火墙规则。

注意：

> 没有 UFW 不代表系统绝对没有其他防火墙，因此后续仍可能需要检查 nftables / iptables。

---

# 十五、第十二步：测试 VPS 内部 SSH 是否正常

执行：

```
ssh-keyscan -T 5 127.0.0.1
```

成功返回：

```
127.0.0.1:22 SSH-2.0-OpenSSH_9.6p1 Ubuntu-3ubuntu13.16
```

并返回 SSH Host Key。

这一步非常关键。

因为它证明：

```
Ubuntu 本机
 ↓
127.0.0.1:22
 ↓
sshd
 ↓
SSH 协议握手 ✅
```

所以：

> SSH 服务自身是可以正常完成 SSH 握手的。

---

# 十六、第十三步：Windows SSH 仍然失败

Windows：

```
ssh -vvv -o PubkeyAuthentication=no -o PreferredAuthentications=password -o ConnectTimeout=10 root@107.150.26.37
```

最终：

```
Local version string SSH-2.0-OpenSSH_for_Windows_9.5
kex_exchange_identification: Connection closed by remote host
Connection closed by 107.150.26.37 port 22
```

说明：

```
TCP：✅
SSH 端口：✅
SSH 本机服务：✅
Windows SSH：❌
```

而且还没有进入：

```
root@107.150.26.37's password:
```

所以不是 root 密码错误。

---

# 十七、第十四步：使用 tcpdump 进一步定位

VPS 上执行：

```
tcpdump -ni any 'tcp port 22'
```

进入监听状态。

同时 Windows 发起：

```
ssh root@107.150.26.37
```

当时的现象是：

```
Windows SSH：
Connection closed by 107.150.26.37 port 22
```

而对应外部 SSH 数据包没有像预期一样进入 Ubuntu 的正常 SSH 处理流程。

结合前面：

```
127.0.0.1 SSH ✅
0.0.0.0:22 ✅
sshd ✅
ssh.socket ✅
Windows TCP ✅
```

问题范围进一步缩小到：

```
Windows 本机网络栈
Clash / Meta 虚拟网卡
路由
网络过滤层
```

---

# 十八、最终发现：Clash Verge 虚拟网卡

之前 Windows 网络测试出现：

```
InterfaceAlias : Meta
```

说明 Clash/Mihomo 虚拟网络接口曾经参与网络流量处理。

之后进行了关键的对照实验：

## 原状态

```
Clash Verge 虚拟网卡存在
↓
SSH 无法连接 VPS
```

## 删除虚拟网卡

直接删除 Clash Verge 虚拟网卡。

结果：

```
SSH 立即恢复正常
```

## 重新安装虚拟网卡

再次安装：

```
Clash Verge 虚拟网卡
```

结果：

```
SSH 依然能够正常连接
```

因此最终判断为：

> **不是“虚拟网卡本身不能和 SSH 共存”，而是此前 Clash Verge 虚拟网卡相关的网络状态存在异常。**

可能涉及：

```
旧虚拟网卡状态
路由表
接口优先级
TUN 状态
Windows 网络过滤状态
Clash 服务状态
```

删除虚拟网卡后，相当于对这部分网络状态进行了重置。

重新安装之后：

```
重新创建虚拟网卡
重新建立路由/网络状态
```

因此网络恢复正常。

目前没有证据能够进一步确定具体是哪一个 Windows 网络组件损坏，所以不要武断地说一定是某一条路由或者某一个 WFP 规则。

---

# 十九、最终问题定位结论

本次问题最终定位：

```
DediRock VPS
    ✅ 正常

Ubuntu
    ✅ 正常

sshd
    ✅ 正常

ssh.socket
    ✅ 正常

22端口
    ✅ 正常

Fail2ban
    ✅ 当前没有封禁 IP

UFW
    ✅ 未安装

SSH配置
    ✅ 正常

VPS内部SSH
    ✅ 正常

Windows → VPS TCP
    ✅ 正常

Clash Verge Meta/TUN 网络层
    ⚠️ 曾出现异常

最终解决
    ✅ 删除 Clash 虚拟网卡
    ✅ 重新安装虚拟网卡
    ✅ SSH 恢复正常
```

---

# 二十、以后再遇到类似问题，推荐排查顺序

不要一开始就重装 VPS 或修改 SSH 配置。

推荐按照下面顺序：

```
① VPS 后台是否 Online
        ↓
② Windows 是否能 TCP 连 22
        ↓
③ 是否经过 Clash Meta/TUN
        ↓
④ DediRock Web Console 登录 Ubuntu
        ↓
⑤ systemctl status ssh
        ↓
⑥ ss -lntp | grep ':22'
        ↓
⑦ sshd -t
        ↓
⑧ ssh-keyscan 127.0.0.1
        ↓
⑨ Fail2ban / 防火墙
        ↓
⑩ 必要时 tcpdump
        ↓
⑪ 检查 Windows Clash/虚拟网卡
```

---

# 二十一、以后最常用的快速检查命令

## Windows

检查 VPS 22：

```
Test-NetConnection 107.150.26.37 -Port 22
```

检查是否被 Clash 接管：

```
Get-NetAdapter
```

查看路由：

```
route print
```

SSH 调试：

```
ssh -vvv root@107.150.26.37
```

---

## VPS

检查 SSH：

```
systemctl status ssh
```

检查 22：

```
ss -lntp | grep ':22'
```

检查配置：

```
sshd -t
```

检查 socket：

```
systemctl status ssh.socket --no-pager
```

测试本地 SSH：

```
ssh-keyscan -T 5 127.0.0.1
```

检查 Fail2ban：

```
fail2ban-client status sshd
```

查看 SSH 日志：

```
journalctl -u ssh -f
```

抓 22 端口：

```
tcpdump -ni any 'tcp port 22'
```

---

# 二十二、Clash Verge 使用注意事项

## 日常普通代理

推荐：

```
系统代理：开启
TUN：关闭
代理模式：规则
```

适合：

```
浏览器
Git
VSCode
普通 HTTP/HTTPS 应用
```

---

## 需要接管更多程序

例如：

```
某些游戏
UDP 程序
不遵守系统代理的软件
```

再考虑：

```
TUN：开启
```

---

# 二十三、系统代理和 TUN 的核心区别

可以记住：

```
系统代理
=
应用主动使用 Clash
```

而：

```
TUN / 虚拟网卡
=
系统把网络流量送进 Clash
```

因此：

```
系统代理
↓
依赖应用支持系统代理
```

而：

```
TUN
↓
可以接管更多没有代理设置的程序
```

两者进入 Clash 的方式不同。

---

# 二十四、系统代理 / TUN 与规则 / 全局 / 直连的关系

这两个维度不要混淆。

```
                Clash Verge
                     │
          ┌──────────┴──────────┐
          │                     │
       流量入口              路由策略
          │                     │
    ┌─────┴─────┐        ┌──────┼──────┐
    │           │        │      │      │
 系统代理       TUN      规则    全局    直连
    │           │
    └─────┬─────┘
          ↓
       Clash 内核
```

也就是：

```
系统代理 / TUN
=
“流量怎么进入 Clash”
```

而：

```
规则 / 全局 / 直连
=
“进入 Clash 后怎么处理”
```

---

# 二十五、这次问题最重要的经验

以后看到：

```
FinalShell 连不上
SSH Connection closed
```

不要直接认为：

```
VPS 挂了
```

先区分：

```
TCP 不通
```

还是：

```
TCP 通，但 SSH 握手失败
```

再区分：

```
VPS 内部 SSH 服务
```

还是：

```
Windows 本机代理/TUN/虚拟网卡
```

本次最终就是：

```
VPS 本身正常
↓
SSH 服务正常
↓
Windows 网络异常
↓
Clash Verge 虚拟网卡状态异常
↓
删除并重新创建虚拟网卡
↓
恢复正常
```

---

# 二十六、当前状态

```
VPS:
  Provider: DediRock
  Region: Los Angeles
  IP: 107.150.26.37
  OS: Ubuntu 24.04.4 LTS
  Status: Online

SSH:
  Port: 22
  Service: 正常
  Socket: 正常
  Config: 正常

Windows:
  Clash Verge: 正常
  Meta 虚拟网卡: 已重新安装
  SSH: 已恢复正常

最终结论:
  根因方向: Clash Verge / Meta 虚拟网卡网络状态异常
  处理方式: 删除虚拟网卡 → 重新安装虚拟网卡
  当前状态: 正常
```

---

## 后续继续 VPS 配置

SSH 问题解决后，再继续之前没有完成的部分：

```
Clash Verge
    ↓
3X-ui Subscription Server
    ↓
2096
    ↓
/clash/
    ↓
Sub ID
    ↓
Clash YAML
    ↓
成功导入 Clash Verge
```

当前 3X-ui 节点仍然是：

```
Protocol: VMess
Transport: WebSocket
TLS: Enabled
Port: 443
Host: rcardoshaw.de5.net
WS Path: /ricardo
```

订阅服务：

```
Port: 2096
Clash URI Path: /clash/
```

之后继续排查订阅问题时，**不要再把 SSH/TUN 问题和 2096 订阅问题混在一起。**

## 订阅链接导入问题

感觉这也还是个大问题，好像根本原因是因为我的订阅链接浏览器输入并没有显示yaml，而是卡住