
# 第 3 章 · 排查对等体问题

> **本章涵盖**：BGP 对等体建立的四段过程与每段失败的判据；对等体问题的六大分类；TCP 179 连不上的五类根因（未监听、ACL、源地址、多跳 TTL、MTU）；OPEN 与能力协商失败（AS 号、router-id、4 字节 AS、认证、hold time、AFI/SAFI 未激活）；会话抖动与资源耗尽（hold timer、max-prefix、BFD、CoPP）；IOS / IOS XR / NX-OS 命令对照；**六个三平台实战用例**。

---

## 3.1 对等体是怎么"起来"的：四段式

BGP 会话不是一条命令配完就有的，它要走过四段。排错就是把"卡在哪一段"先定位出来，**不要一上来就 debug**。

| 段 | 发生什么 | 失败时 `summary` 显示 | 典型根因 |
|----|---------|---------------------|---------|
| ① TCP 连接 | 主动方发起 SYN 到对端 179，完成三次握手 | `Active` / `Connect` / `Idle` | 没配邻居、ACL 拦 179、源地址不对、多跳 TTL=1、路由不可达 |
| ② OPEN 交换 | 双方互换 ASN、router-id、hold time、能力 | `OpenSent` / `OpenConfirm` | AS 号不匹配、router-id 冲突、hold time 不可接受、认证失败 |
| ③ KEEPALIVE | 各自回一个 KEEPALIVE，会话转 Established | `OpenConfirm` | 报文被丢（MTU、CoPP、单向丢包） |
| ④ 能力/地址族协商 | 确认 AFI/SAFI 已激活、能力集交集 | `Established` 但 `PfxRcd = 0` | 未 `activate`、AF 不匹配、XR 未关联 route-policy |

> **关键**：`Established` 只是第③段通过，**不代表能交换路由**。大量工单是"起来了但没路由"，那是第④段或第 4 章的内容。

### 3.1.1 一张判据表：用一条命令定段

| 你要回答的问题 | 命令（三平台通用写法） | 判据 |
|--------------|---------------------|------|
| TCP 通了吗 | `show tcp brief \| include 179` | 有 `ESTABLISHED` → 过了①段 |
| 对端在监听吗 | `telnet <peer> 179` | 立刻 `Connection refused` → 没监听；挂住 → 在监听 |
| 卡在哪个 FSM 状态 | `show bgp ipv4 unicast summary` | `St/PfxRcd` 列是文字还是数字 |
| 为什么掉的 | `show bgp ipv4 unicast neighbors <ip>` | `Last reset ... due to` |
| 收到了什么 NOTIFICATION | `show logging` | `%BGP-3-NOTIFICATION` 行 |
| AF 激活了吗 | `show bgp ipv4 unicast neighbors <ip>` | `Address family IPv4 Unicast: advertised and received` |

### 3.1.2 NOTIFICATION 错误码：会话是被谁"踢"掉的

会话建立后若被对端主动关闭，一定伴随一条 NOTIFICATION。记住六个主码，比背配置有用：

| 码 | 名称 | 常见子码 / 含义 |
|----|------|----------------|
| 1 | Message Header Error | 报文长度/类型异常（MTU 截断、版本不匹配） |
| 2 | OPEN Message Error | **2 = Bad Peer AS**、3 = Bad BGP Identifier、4 = Unsupported Optional Parameter、6 = Unacceptable Hold Time |
| 3 | UPDATE Message Error | 属性格式/长度错误（malformed attribute list、invalid origin） |
| 4 | Hold Timer Expired | 三个 hold time 内没收到 KEEPALIVE（单向丢包、CPU 挂死） |
| 5 | FSM Error | 状态机收到不该收的报文（通常伴随 Bug 或严重丢包） |
| 6 | Cease | **1 = max prefixes**、2 = admin shutdown、4 = peer de-configured、6 = other |

```
! IOS
%BGP-3-NOTIFICATION: sent to neighbor 10.1.12.2 4/0 (hold time expired) 0 bytes

! IOS XR
%ROUTING-BGP-5-ADJCHANGE : neighbor 10.1.12.1 Down - BGP Notification sent, hold time expired

! NX-OS
%BGP-5-ADJCHANGE: neighbor 10.1.12.2 Down BGP Notification sent
%BGP-3-NOTIFICATION: sent to neighbor 10.1.12.2 4/0 (hold time expired) 0 bytes
```

> **判据**：`sent` 说明是**本机**发出去的（本机认为对端死了）；`received` 说明是**对端**关的你。方向搞反，根因就查错设备了。

---

## 3.2 对等体问题的六大分类

工单里的"邻居起不来"其实是六种完全不同的病，混在一起查必然低效。

| 类别 | 典型现象 | 首选判据 | 主要排查方向 |
|------|---------|---------|------------|
| ① 从未建立 | 一直 `Active` / `Idle` | `show tcp brief` 无 179 | TCP 层：配置、ACL、源地址、路由、TTL |
| ② OPEN 被拒 | `OpenSent` 反复 reset | `show logging` 的 NOTIFICATION 2/x | ASN、router-id、认证、hold time |
| ③ 建了但无 AF | `Established` + `PfxRcd 0` | `neighbors <ip>` 的能力段 | `activate`、AF 配置、route-policy |
| ④ 周期性抖动 | `Up/Down` 时间短、反复 ADJCHANGE | `Last reset ... due to` | hold timer、MTU、链路误码、CPU、max-prefix |
| ⑤ 单向可达 | A 侧 Established、B 侧 Active | 两侧 `show tcp brief` 对比 | 非对称路由、ACL 方向、uRPF |
| ⑥ 资源/平台 | 邻居数上不去、内存告警 | `show bgp summary` + `show memory` | max-prefix、update-group、CoPP、TCAM |

> 生产排错顺序：**先分类（30 秒）→ 再定段（2 分钟）→ 最后看细节**。跳过前两步直接 `debug` 是最常见的低效动作。

---

## 3.3 第一段：TCP 179 连不上

现象固定：`show bgp ipv4 unicast summary` 里一直是 `Active`，`show tcp brief` 看不到 179。

### 3.3.1 先证明"到底是谁的问题"

```
! 主动方（IOS）：看本机有没有发出 SYN、有没有半开连接
show tcp brief | include 179
（无输出 = 连 SYN-SENT 都没有，本机根本没发起或立即失败）

! 换源地址测试（关键：BGP 用的是哪个源地址，测试就要用哪个）
telnet 10.1.12.2 179 /source-interface GigabitEthernet0/1
Trying 10.1.12.2, 179 ...
% Connection refused by remote host        ← 对端可达但 179 没监听
% Connection timed out                     ← 报文被丢（ACL/路由/TTL）
（挂住不返回，最后被对端断开）              ← 正常，对端在监听
```

```
! XR / NX-OS 无 source-interface 参数，用 ping 验证源地址 + telnet 验证端口
ping 10.1.12.2 source Loopback0 count 3
telnet 10.1.12.2 179
```

> `telnet <ip> 179` 是排 BGP 对等体最有价值的一条命令：**立刻 refused = 对端缺配置；超时 = 中间被拦**。两者根因完全不同，一分钟就能分开。

### 3.3.2 根因一：对端根本没监听（缺配置 / passive / VRF 放错）

```
! 主动方 R1 (IOS)
show bgp ipv4 unicast summary
10.1.12.2       4 65200       1       0        0    0    0 00:03:12     Active

! 对端 R2 (IOS XR) 上核对
show running-config router bgp 65200 neighbor 10.1.12.1
% No such configuration item(s)          ← 根因：对端没有这个邻居
```

**三平台的"监听"检查**

```
! IOS：BGP 是否在监听 179（有邻居配置才会监听，除非配了 listen range）
show control-plane host open-ports | include 179
    179  tcp  bgp                 LISTEN

! XR
show tcp brief | include 179

! NX-OS：同样有 open-ports（有的版本在 show sockets 下）
show control-plane host open-ports
```

**被动模式（只接受、不发起）**——三平台关键字一致：

```
! IOS
router bgp 65100
 neighbor 10.1.12.2 transport connection-mode passive

! IOS XR
router bgp 65200
 neighbor 10.1.12.1
  transport connection-mode passive

! NX-OS
router bgp 65300
  neighbor 10.1.23.2
    transport connection-mode passive
```

> 配了 `passive` 后，本机不发起连接，**必须由对端主动**。两侧都配 passive = 永远 `Idle`，这是非常常见的"人为死锁"。

**VRF 放错**：邻居配在了 `vrf A`，报文从 `vrf B` 的接口进来，BGP 进程按 VRF 各自维护 socket，照样不通。

```
! IOS
show bgp vrf all summary            ! 逐个 VRF 找这个邻居在哪
! XR
show bgp vrf all summary
! NX-OS
show bgp all summary | include 10.1.12.2
```

### 3.3.3 根因二：ACL / 防火墙把 179 拦了

```
! 接口入向 ACL（IOS），漏了 179 是新人最常犯的错
ip access-list extended ACL-WAN-IN
 permit tcp any host 203.0.113.1 eq 179
 permit tcp host 203.0.113.1 eq 179 any     ! 别忘反向：BGP 源端口也可以是 179
 permit icmp any any
 deny   ip any any log
!
interface GigabitEthernet0/0
 ip access-group ACL-WAN-IN in
```

```
! NX-OS 用 ACL + 计数器确认命中
show ip access-lists ACL-WAN-IN
        permit tcp any host 203.0.113.1 eq bgp  (12 matches)   ← 有匹配
        deny   ip any any log  (3812 matches)                  ← 大量被拒，高度可疑
```

```
! XR：ACL 在接口下用 ipv4 access-group
show ipv4 access-lists ACL-WAN-IN hardware ingress location 0/0/CPU0
```

**判据**：ACL 的 `deny` 条目计数在**持续增长**，而 BGP 会话起不来，基本可以锁定。注意 BGP 可能只丢掉 SYN-ACK（单向拦截），所以两侧都要看。

> 防火墙场景额外注意两点：① 会话表老化时间要大于 BGP 的 KEEPALIVE 间隔（默认 60s，防火墙 idle timeout 常配 30 分钟，一般够）；② 状态检测设备必须放行双向 179，不能只放行"内到外"。

### 3.3.4 根因三：源地址不对（update-source 缺失）

BGP 用**出接口地址**作为默认源。对端只认"配置里的那个邻居地址"，源地址一错，SYN 到了也会被丢（因为对端找不到匹配的邻居条目）。

```
! R1 用物理口地址发起，但对端配的邻居是 Lo0
show bgp ipv4 unicast neighbors 10.1.12.2 | include "Local host|Foreign host"
  Local host: 10.1.12.1, Local port: 179
  Foreign host: 10.1.12.2, Foreign port: 52411
```

```
! 修复：三平台 update-source 位置不同（XR 在 neighbor 下，不在 AF 下）
! IOS
neighbor 192.168.2.2 update-source Loopback0
! XR
neighbor 192.168.2.2
 update-source Loopback0
! NX-OS
neighbor 192.168.2.2
  update-source loopback0
```

**验证源地址是否真的生效**（会话起来后）：

```
show tcp brief | include 179
...     192.168.1.1:179     192.168.2.2:38876   ESTABLISHED   ← Local 必须是 Lo0
```

> 用 Lo0 建邻还必须同时满足三条：① 双方到对端 Lo0 有路由；② 双方都配 `update-source`；③ EBGP 需 `ebgp-multihop`（TTL 默认 1，跨一跳就死）。

### 3.3.5 根因四：多跳 EBGP 的 TTL

```
! 默认 EBGP 报文的 TTL = 1，跨过任何一台设备就被丢弃，现象是 Connect → Active 循环
! IOS
neighbor 192.168.2.2 ebgp-multihop 3
! XR（在 AF 下）
neighbor 192.168.2.2
 address-family ipv4 unicast
  ebgp-multihop 3
! NX-OS（在 AF 下）
neighbor 192.168.2.2
  address-family ipv4 unicast
    ebgp-multihop 3
```

**更安全的替代：GTSM（TTL 安全）**，要求报文 TTL 必须 ≥ 255-hops+1，天然抗伪造：

```
! IOS
neighbor 203.0.113.9 ttl-security hops 1
! XR
neighbor 203.0.113.9
 address-family ipv4 unicast
  ttl-security
! NX-OS
neighbor 203.0.113.9
  address-family ipv4 unicast
    ttl-security hops 1
```

> `ebgp-multihop` 和 `ttl-security` 互斥。用 GTSM 时，若中间设备改写了 TTL（某些防火墙/NAT），会话也起不来——这是 GTSM 部署后"突然不通"的经典原因。

### 3.3.6 根因五：路径 MTU 导致 OPEN 之后反复断

TCP 能握上手，但在 OPEN/KEEPALIVE 阶段反复 `OpenSent → Active`，或者 Established 后每次发大 UPDATE 就掉。

```
! 验证：带 DF 位打大包
ping 10.1.12.2 size 1500 df-bit repeat 3
!!!!!                                       ← 通，MTU 没问题
.....                                       ← 不通，链路 MTU < 1500
```

```
! 三平台的 PMTUD / MSS 调整
! IOS
ip tcp path-mtu-discovery                    ! 默认开启
interface GigabitEthernet0/0
 ip tcp adjust-mss 1400
! XR
tcp path-mtu-discovery
interface GigabitEthernet0/0/0/0
 ipv4 tcp-mss-adjust 1400
! NX-OS
ip tcp path-mtu-discovery
interface Ethernet1/1
 ip tcp adjust-mss 1400
```

> BGP 单个 UPDATE 最大可达 4096 字节（Extended Message 能力），一旦路径 MTU 偏小且 PMTUD 被 ICMP 过滤阻断，就会出现"小 UPDATE 能过、大 UPDATE 丢"的诡异现象——表现为**路由部分缺失，且每次缺失的条目不固定**。

---

## 3.4 第二段：OPEN 被拒（会话起来了又立刻掉）

现象：`summary` 里状态在 `OpenSent` / `OpenConfirm` / `Established` 之间跳，`Up/Down` 只有几秒，日志里成对出现 `Down` 与 `NOTIFICATION`。

**通用排查动作**：先看清是"发"还是"收"。

```
! IOS
show bgp ipv4 unicast neighbors 10.1.12.2 | include "Last reset|Notification|state"
  BGP state = Active
  Last reset 00:00:12, due to BGP Notification received, Bad Peer AS
  Notification received: 2/2 (BGP error code 2, subcode 2)

! XR
show bgp ipv4 unicast neighbors 10.1.12.1 | include "Last reset|Notification|state"
  Last reset: 00:00:12, due to BGP Notification received
  Notification: 2/2 (BGP error code 2, subcode 2)

! NX-OS
show bgp ipv4 unicast neighbors 10.1.23.2 | include "Last reset|Notification|BGP state"
  BGP state = Active
  Last reset 00:00:12, due to BGP Notification received
  Notification received: 2/2 (BGP error code 2, subcode 2)
```

> 三平台字段名略有差异，但都必带 **`Last reset ... due to`** 和 **`Notification`** 两行。这两行是第②段的全部答案。

### 3.4.1 根因一：AS 号不匹配（NOTIFICATION 2/2）

`Bad Peer AS` 是"我以为你是 AS X，你在 OPEN 里说你是 AS Y"。

```
! 对端实际 AS 与本地 remote-as 不一致
show bgp ipv4 unicast neighbors 10.1.12.2 | include "remote AS|local AS"
  Local AS: 65100, Local router ID: 192.168.1.1
  4-byte AS: 65100
  remote AS: 65200

! XR 上看对端送来的 OPEN 里写的是什么
show bgp ipv4 unicast neighbors 10.1.12.1 | include "remote AS|Peer AS"
```

**三处最常见的写法错误**

| 错误 | 现象 | 修正 |
|------|------|------|
| 对端 `remote-as` 写成了自己的 AS（应为对端 AS） | 2/2 | 改成对端 ASN |
| 多台设备共用模板，`remote-as` 未按邻居区分 | 2/2 | 用 `peer-group`/`session-group` 统一，但按需覆盖 |
| 4 字节 ASN 用了错误表示法（`65536` vs `1.0`） | 协商异常 / AS_TRANS | 统一用 asplain（`65536`），或全链路启用 asdot |

**4 字节 AS 的特殊点**：当 `ASN > 65535`，与只支持 2 字节的老设备对接时，OPEN 中会送 `AS_TRANS (23456)`，真实 ASN 放在 `AS4_PATH` 属性里。

```
show bgp ipv4 unicast 172.16.1.0/24 | include "AS path"
  AS path: 23456 65100 i            ← 中间有老设备，看到 AS_TRANS
show bgp ipv4 unicast 172.16.1.0/24 | include "AS4"
  AS4 path: 100000 65100 i          ← 真实 ASN 在这里
```

### 3.4.2 根因二：router-id 冲突（NOTIFICATION 2/3）

两台设备的 BGP router-id 相同，双方都会发 `Bad BGP Identifier`（2/3），会话反复 reset。

```
show bgp ipv4 unicast neighbors 10.1.12.2 | include "router ID"
  Local router ID: 192.168.1.1
  Remote router ID: 192.168.1.1          ← 与本地相同！
```

**根因**：router-id 选取顺序是 **手工配置 → 最大的 Lo0 地址 → 最大物理口地址**。克隆配置、批量刷机、VRF 里漏配都会撞。

```
! 三平台显式指定（必配）
! IOS
router bgp 65100
 bgp router-id 192.168.1.1
! XR
router bgp 65200
 bgp router-id 192.168.2.2
! NX-OS
router bgp 65300
  router-id 192.168.3.3
```

> router-id 冲突**跨 VRF 不冲突**（BGP 实例独立），但在同一实例的多条邻居上会全线抖动。改 router-id 会重置所有邻居，务必在窗口内做。

### 3.4.3 根因三：hold time 不可接受（NOTIFICATION 2/6）

hold time 小于 3 秒会被对端以 `Unacceptable Hold Time` 拒绝。

```
! 三平台的计时器配置（单位：秒）
! IOS
neighbor 10.1.12.2 timers 10 30
! XR
neighbor 10.1.12.1
 timers 10 30
! NX-OS
neighbor 10.1.23.2
  timers 10 30
```

**协商规则**：双方取 **较小的 hold time**，KEEPALIVE = hold/3。

```
show bgp ipv4 unicast neighbors 10.1.12.2 | include "hold time|keepalive interval"
  Hold time is 30, keepalive interval is 10 seconds
  Configured hold time is 30, keepalive interval is 10 seconds
```

> 默认 180/60。**不要为了"快速检测"把 hold time 调到 3 秒**——BGP 抖动往往源于短计时器，而不是解决了它。

### 3.4.4 根因四：MD5 认证不匹配

严格说这是 TCP 层失败（BGP 起不来，不是被 NOTIFICATION 踢），现象是 `Connect`/`Active` 循环，日志有 `BADAUTH`。

```
! IOS
neighbor 10.1.12.2 password Cisco123
! XR
neighbor 10.1.12.1
 password clear Cisco123
! NX-OS
neighbor 10.1.23.2
  password 3 Cisco123
```

```
! 判据
%TCP-6-BADAUTH: No MD5 digest from 10.1.12.2(179) to 10.1.12.1(38876)
%TCP-6-BADAUTH: Invalid MD5 digest from 10.1.12.2(179) to 10.1.12.1(38876)
```

| 日志 | 含义 |
|------|------|
| `No MD5 digest` | **对端没配**密码 |
| `Invalid MD5 digest` | 两边都配了，但**密码不一致** |

> 现代部署建议改用 **TCP-AO**（XR/NX-OS 支持，IOS 部分平台支持），MD5 已被认为不安全且对 CPU 有额外开销。

### 3.4.5 根因五：AFI/SAFI 未激活（Established 但 PfxRcd = 0）

这是第④段失败，`summary` 看起来"完全正常"，只是前缀数永远是 0。

```
show bgp ipv4 unicast summary
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.1.12.2       4 65200      42      40        0    0    0 00:31:02          0
                                                                          ↑ 0 且 Up/Down 很长 = AF 没激活
```

```
! 判据：能力段里没有 "received"
show bgp ipv4 unicast neighbors 10.1.12.2 | include "Address family"
  Address family IPv4 Unicast: advertised          ← 只有 advertised，没有 received
```

**三平台激活写法**

```
! IOS：默认激活 IPv4，但一旦用了 no bgp default ipv4-unicast 就必须显式 activate
router bgp 65100
 address-family ipv4
  neighbor 10.1.12.2 activate

! XR：AF 在 neighbor 下显式开，且必须关联 route-policy 才会收/发
router bgp 65200
 neighbor 10.1.12.1
  address-family ipv4 unicast
   route-policy PASS-ALL in
   route-policy PASS-ALL out

! NX-OS：邻居下必须再进 AF（最容易漏）
router bgp 65300
  neighbor 10.1.23.2
    remote-as 65200
    address-family ipv4 unicast
```

> **XR 的静默陷阱**：`neighbor ... address-family ipv4 unicast` 里不写 `route-policy in` 时，入向路由**全部丢弃且不告警**；不写 `route-policy out` 时，出向**一条都不发**。会话是 Established，`PfxRcd` 为 0，日志干干净净——跨厂商对接第一大坑。

---

## 3.5 会话抖动与资源耗尽

"稳不住"比"起不来"更难查，因为故障往往在两次观测之间就恢复了。

### 3.5.1 抖动必须先量化，再归因

```
! 第一步：看掉了多少次、什么原因（三平台通用字段）
show bgp ipv4 unicast neighbors 10.1.12.2 | include "Last reset|reset|down"
  Last reset 00:02:41, due to BGP Notification received, hold time expired
  Connections established 27; dropped 26
  Last reset 00:02:41, due to hold timer expired

! 第二步：看抖动是否有规律
show logging | include "ADJCHANGE|BGP-5|BGP-3"
00:02:41: %BGP-5-ADJCHANGE: neighbor 10.1.12.2 Down BGP Notification received
00:07:41: %BGP-5-ADJCHANGE: neighbor 10.1.12.2 Up
```

| `due to` 的值 | 含义 | 主力方向 |
|--------------|------|---------|
| `hold timer expired` | 3 个 hold time 没收到 KEEPALIVE | 单向丢包、中间设备、CPU |
| `BGP Notification received` | 对端主动关 | 看 NOTIFICATION 码（2/x、6/x） |
| `BGP Notification sent` | 本机关掉对端 | 本机的 max-prefix / 策略 / hold |
| `Administrative reset` | 有人 `clear` | 变更记录、脚本 |
| `Interface flap` / `TCP reset` | 链路或会话表 | 物理层、防火墙老化 |
| `Peer closed the session` | 对端进程重启 | 对端 `show version` / uptime |

> **判据铁律**：`dropped` 与 `established` 接近，说明**每次起来很快就掉**；两者都很大（如 200/150）说明**长期不稳定**。前者查 OPEN 层，后者查资源与链路。

### 3.5.2 hold timer expired 的三种真实原因

```
! ① 单向丢包：A 能到 B，B 回不来（非对称路由 / uRPF / ACL）
! 在 A 上连续看计数器，KEEPALIVE 只增不减
show bgp ipv4 unicast neighbors 10.1.12.2 | include "MsgRcvd|MsgSent"
  Message received: 1204, sent: 2411        ← 严重不对称
```

```
! ② CPU/CoPP 把 BGP 报文限速丢掉了
! IOS
show policy-map control-plane
  Service-policy input: COPP
    Class-map: class-default (match-any)
      police cir 1000000 conform-action transmit exceed-action drop
! XR
show policy-map interface all | include "police|dropped"
! NX-OS
show policy-map interface control-plane
```

```
! ③ 链路误码 / 光衰（表现为无规律的 30~180 秒抖动）
show interfaces GigabitEthernet0/1 | include "error|CRC|input errors"
     0 input errors, 0 CRC, 0 frame, 0 overrun, 0 ignored
```

### 3.5.3 max-prefix：对端"被自己踢掉"

```
! IOS
neighbor 10.1.12.2 maximum-prefix 10000 80 warning-only
! XR
neighbor 10.1.12.1
 address-family ipv4 unicast
  maximum-prefix 10000 80 warning-only
! NX-OS
neighbor 10.1.23.2
  address-family ipv4 unicast
    maximum-prefix 10000 80 warning-only
```

```
! 达到阈值（默认 75% 告警，100% 断会话）
%BGP-3-MAXPFX: Number of prefixes received from 10.1.12.2 (10000) reaches maximum 10000 for peer
%BGP-5-ADJCHANGE: neighbor 10.1.12.2 Down BGP Notification sent
! 且 NOTIFICATION 为 6/1 (Cease / Maximum Number of Prefixes Reached)
```

> 生产建议一律带 `warning-only` 先观察一个周期，确认正常水位后再收紧。`maximum-prefix` 踢掉的是**整个会话**，代价是全部路由消失——比"多收几条路由"严重得多。

### 3.5.4 BFD：把"分钟级"收敛压到"毫秒级"，也会放大抖动

```
! IOS
interface GigabitEthernet0/1
 bfd interval 300 min_rx 300 multiplier 3
router bgp 65100
 neighbor 10.1.12.2 fall-over bfd

! XR
router bgp 65200
 neighbor 10.1.12.1
  bfd fast-detect
  bfd multiplier 3
  bfd minimum-interval 300

! NX-OS
feature bfd
interface Ethernet1/1
  bfd interval 300 min_rx 300 multiplier 3
router bgp 65300
  neighbor 10.1.23.2
    bfd
```

> BFD 的代价：链路轻微误码就会触发 BGP 重置。光纤质量差的环境里，BFD 间隔不要小于 300ms×3。

### 3.5.5 资源耗尽：邻居数上不去 / 会话无故重置

```
! 三平台看 BGP 进程的资源占用
! IOS
show processes cpu | include BGP
show memory statistics | include BGP
! XR
show processes bgp | include "CPU|Memory"
! NX-OS
show processes cpu | include bgp
show system resources
```

| 现象 | 判据 | 处理 |
|------|------|------|
| 邻居上限 | `show bgp summary` 到某个数就不再加 | 平台 BGP peer scale 限制，查 `show bgp process`/版本 release notes |
| 会话莫名 reset 且无 NOTIFICATION | CPU 中断级占用高 | 查 CoPP、SPAN、debug 残留 |
| 内存告警后邻居批量 down | `show memory` 持续下降 | 查 max-prefix、路由泄漏、update-group 膨胀 |

> **别忘了 `debug` 残留**：一个忘记关的 `debug ip bgp updates` 就能让 CPU 打满、邻居批量超时。这是"昨天还好、今天全军覆没"的经典原因，先 `show debug` 再谈其他。

---

## 3.6 三平台命令对照（对等体排错专用）

| 目的 | IOS | IOS XR | NX-OS |
|------|-----|--------|-------|
| 邻居总览 | `show bgp ipv4 unicast summary` | 同左 | 同左 |
| 邻居详情 | `show bgp ipv4 unicast neighbors IP` | 同左 | 同左 |
| TCP 179 连接 | `show tcp brief \| include 179` | `show tcp brief` | `show tcp brief` |
| 监听端口 | `show control-plane host open-ports` | `show tcp brief` | `show control-plane host open-ports` |
| 邻居配置核对 | `show run \| section router bgp` | `show running-config router bgp X neighbor Y` | `show run bgp` |
| 日志 | `show logging` | `show logging` | `show logging log` |
| FSM 事件 debug | `debug ip bgp events` | `debug bgp neighbor-events` | `debug bgp events` |
| UPDATE debug | `debug ip bgp updates` | `debug bgp update` | `debug bgp updates` |
| TCP 层 debug | `debug ip tcp transactions` | `debug tcp` | `debug tcp` |
| 内置抓包 | 无（需 SPAN/ERSPAN） | `monitor capture` | `ethanalyzer local interface ...` |
| 软重置 | `clear ip bgp IP soft in` | `clear bgp ipv4 unicast IP soft in` | `clear bgp ipv4 unicast IP soft in` |
| 硬重置 | `clear ip bgp IP` | `clear bgp ipv4 unicast IP` | `clear bgp ipv4 unicast IP` |
| 优雅重启 | `bgp graceful-restart` | `bgp graceful-restart` | `graceful-restart` |
| 清除前先留证 | `show tech-support bgp` | `show tech-support bgp` | `show tech-support bgp` |

> **NX-OS 特有坑**：`clear bgp ipv4 unicast *` 与 `clear bgp ipv4 unicast IP` 都**不会**重建 TCP，若怀疑 TCP 层异常（如会话表卡死），要先 `clear tcp connection` 或干脆 `no feature bgp` 再开。

---

## 3.7 三平台实战用例

> 输出为示意（字段顺序与真实设备一致）。三平台指 **IOS**、**IOS XR**、**NX-OS**。

### 拓扑

```
        AS 65100                AS 65200                      AS 65200
   R1 (IOS) ──────── R2 (IOS XR) ──────── R3 (NX-OS) ──────── R4 (IOS)
                10.1.12.0/30          10.1.23.0/30
   Gi0/1 10.1.12.1     10.1.12.2 Gi0/0/0/1
                       10.1.23.2 Gi0/0/0/2   10.1.23.3 Eth1/1
   Lo0 192.168.1.1/32  Lo0 192.168.2.2/32  Lo0 192.168.3.3/32  Lo0 192.168.4.4/32
                        └────── IBGP（Lo0 建邻）──────┘   └── EBGP ──┘
```

| 设备 | 平台 | AS | 本章中的角色 |
|------|------|-----|------------|
| R1 | IOS | 65100 | EBGP 主动方（多个用例的故障侧） |
| R2 | IOS XR | 65200 | ASBR，EBGP + IBGP 汇合点 |
| R3 | NX-OS | 65200 | IBGP 对端（Lo0 建邻） |
| R4 | IOS | 65200 | 内部路由源，验证 AF 激活 |

---

### 用例 1：配置漏了一半——一侧永远 `Active`

**症状**：R1 侧配完 `neighbor 10.1.12.2 remote-as 65200`，R2 侧只做了接口和路由，忘了配 BGP 邻居。R1 的 `Up/Down` 一直显示 `Active`，`MsgRcvd` 停在 1。

**步骤 1 — 一分钟定段（先证明是①段）**

```
! R1 (IOS)
show bgp ipv4 unicast summary
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.1.12.2       4 65200       1       0        0    0    0 00:04:12      Active

show tcp brief | include 179
（无输出 → TCP 都没建起来，第①段失败）
```

**步骤 2 — 用 telnet 分清"没监听"还是"被拦"**

```
! R1
telnet 10.1.12.2 179
Trying 10.1.12.2, 179 ...
% Connection refused by remote host
```

**判据**：`refused` = 报文到得了对端，对端回 RST（179 无监听）；`timed out` = 报文根本没到或没回来。**这一条把"配置问题"和"网络问题"彻底切开**。

**步骤 3 — 到对端验证（三平台）**

```
! R2 (IOS XR)
show running-config router bgp 65200
% No such configuration item(s)       ← BGP 进程都没配

! 若已配进程但漏了邻居
show running-config router bgp 65200 neighbor 10.1.12.1
% No such configuration item(s)
```

```
! 如果是 NX-OS 作为对端（R3 视角）
show run bgp | include "neighbor 10.1.23.2|remote-as"
neighbor 10.1.23.2
  remote-as 65200                      ← 只有 remote-as，没有 address-family
```

**步骤 4 — 补齐并验证**

```
! R2 (XR)
router bgp 65200
 bgp router-id 192.168.2.2
 address-family ipv4 unicast
 !
 neighbor 10.1.12.1
  remote-as 65100
  address-family ipv4 unicast
   route-policy PASS-ALL in
   route-policy PASS-ALL out
  !
 !
!
```

```
! R1 侧 3 秒内应看到状态变化（无需 clear）
show bgp ipv4 unicast summary
10.1.12.2       4 65200       3       4        0    0    0 00:00:05          0

show tcp brief | include 179
...     10.1.12.1:179      10.1.12.2:41217     ESTABLISHED
```

> **注意**：`PfxRcd` 为 0 在这里是正常的——本例只验证会话建立。若 R2 有前缀要送，还得看 3.4.5 的 AF 与 route-policy。

---

### 用例 2：ACL 放行了"内到外"，没放行"外到内"

**症状**：R1、R2 两侧配置都正确，`telnet 179` 超时，会话一直 `Active`。删掉 R2 接口上的 ACL 就恢复。

**步骤 1 — 怀疑 ACL 时先看计数器，不要先删**

```
! R2 (IOS XR)：接口入向 ACL
show ipv4 access-lists ACL-WAN-IN
IPv4 access list ACL-WAN-IN
   10 permit tcp any host 10.1.12.2 eq bgp     (0 matches)
   20 permit icmp any any                      (18 matches)
   30 deny   ipv4 any any log                  (2461 matches)   ← 持续增长
```

```
! NX-OS 同样看 match 计数
show ip access-lists ACL-WAN-IN
        deny ip any any log  (18422 matches)
```

**步骤 2 — 确认被丢的正是 BGP**

```
! IOS / XR 打开 ACL 日志（谨慎，会占 CPU）
show logging | include ACL-WAN-IN
%SEC-6-IPACCESSLOGP: list ACL-WAN-IN denied tcp 10.1.12.1(179) -> 10.1.12.2(41217), 1 packet
```

**根因**：`deny ip any any` 兜底 + 只写了"允许我到对端"，漏了"允许对端到我"。BGP 的 TCP 报文**双向**都需要 179（一端源端口 179、另一端目的端口 179）。

**步骤 3 — 修正 ACL（两个方向都放行）**

```
! IOS
ip access-list extended ACL-WAN-IN
 permit tcp host 10.1.12.1 host 10.1.12.2 eq bgp
 permit tcp host 10.1.12.1 eq bgp host 10.1.12.2
 permit icmp any any
 deny   ip any any log
```

```
! XR
ipv4 access-list ACL-WAN-IN
 10 permit tcp host 10.1.12.1 host 10.1.12.2 eq bgp
 20 permit tcp host 10.1.12.1 eq bgp host 10.1.12.2
 30 permit icmp any any
 40 deny ipv4 any any log
!
interface GigabitEthernet0/0/0/1
 ipv4 access-group ACL-WAN-IN ingress
```

```
! NX-OS
ip access-list ACL-WAN-IN
  10 permit tcp 10.1.12.1/32 10.1.12.2/32 eq bgp
  20 permit tcp 10.1.12.1/32 eq bgp 10.1.12.2/32
  30 permit icmp any any
  40 deny ip any any log
interface Ethernet1/1
  ip access-group ACL-WAN-IN in
```

**步骤 4 — 验证**

```
! R2
show ipv4 access-lists ACL-WAN-IN
   10 permit tcp host 10.1.12.1 host 10.1.12.2 eq bgp  (37 matches)   ← 开始命中

! R1
show bgp ipv4 unicast summary
10.1.12.2       4 65200      12      13        2    0    0 00:00:41          1
```

**四个高频 ACL 事故（都是同一个病）**

| 场景 | 症状 | 修正 |
|------|------|------|
| 只放行单向 179 | telnet 超时 | 两个方向都放行 |
| 只放行 `eq bgp` 目的端口 | 对端主动发起时失败 | 同时放行 `src eq 179` |
| ACL 绑在错误的接口/方向 | 计数器不增长 | `show ip interface` 确认 `Inbound`/`Outbound` |
| 中间的防火墙有独立安全策略 | 设备 ACL 全放行仍不通 | 查防火墙会话表与策略日志 |

> **经验值**：`show ip access-lists` 的 `deny` 计数是排 ACL 问题最快的证据。**计数不增长 = 问题不在这个 ACL 上**，别浪费时间删策略。

---

### 用例 3：用 Lo0 建 IBGP 卡在 `Active`（三个条件缺一不可）

**场景**：R2（XR）与 R3（NX-OS）用 Lo0 建 IBGP，便于多条链路冗余。两侧都写了 `neighbor 192.168.3.3 remote-as 65200`，会话起不来。

**步骤 1 — 先看 TCP 是谁在发芽**

```
! R2 (IOS XR)
show bgp ipv4 unicast summary
192.168.3.3     4 65200       1       0        0    0    0 00:02:07      Active

show tcp brief | include 179
（无 ESTABLISHED）
```

```
! R3 (NX-OS)：看有没有收到半开连接
show tcp brief | include 179
...     10.1.23.3:179     10.1.23.2:39124    SYN_RCVD   ← 收到 SYN，但源地址是物理口！
```

**判据**：`SYN_RCVD` 里对端源地址是 `10.1.23.2`（物理口），说明**对端没配 `update-source`**。R3 的邻居条目是 `192.168.2.2`，找不到匹配项，于是不回 SYN-ACK，连接卡死。

**步骤 2 — 逐条核对三个条件**

| 条件 | 检查命令（三平台） | 本场景结果 |
|------|------------------|-----------|
| ① 到对端 Lo0 有路由 | IOS/XR `show route 192.168.3.3/32`、NX-OS `show ip route 192.168.3.3/32` | ❌ 无路由 |
| ② 双方都配 `update-source` | `show run` 里查 | ❌ 仅 R2 配了 |
| ③ IBGP 不需要 `ebgp-multihop` | — | ✅ iBGP 默认 TTL 255 |

```
! R2 (XR) 确认路由缺失
show route 192.168.3.3/32
% Network not found in table
```

```
! R3 (NX-OS) 确认路由缺失
show ip route 192.168.3.3/32
% No matching route
```

> **注意 ③**：`update-source Loopback0` 只在**双方都有到对端 Lo0 路由**时才有意义。若对端 Lo0 不可达，TCP 连 SYN 都发不出去——先查路由，再查源地址。

**步骤 3 — 补齐（三平台完整配置）**

```
! R2 (IOS XR)
router static
 address-family ipv4 unicast
  192.168.3.3/32 10.1.23.3
 !
!
router bgp 65200
 neighbor 192.168.3.3
  remote-as 65200
  update-source Loopback0          ! 在 neighbor 下，不在 AF 下
  address-family ipv4 unicast
   route-policy PASS-ALL in
   route-policy PASS-ALL out
  !
 !
!
```

```
! R3 (NX-OS)
ip route 192.168.2.2/32 10.1.23.2
!
router bgp 65200
  router-id 192.168.3.3
  neighbor 192.168.2.2
    remote-as 65200
    update-source loopback0
    address-family ipv4 unicast
```

```
! 若对端是 IOS，写法如下
ip route 192.168.3.3 255.255.255.255 10.1.23.3
router bgp 65200
 neighbor 192.168.2.2 remote-as 65200
 neighbor 192.168.2.2 update-source Loopback0
```

**步骤 4 — 验证"源地址真的是 Lo0"**

```
! R2
show tcp brief | include 179
...     192.168.2.2:179     192.168.3.3:44312    ESTABLISHED     ← Local 是 Lo0

show bgp ipv4 unicast neighbors 192.168.3.3 | include "Local host|Foreign host|state"
  BGP state = Established, up for 00:00:23
  Local host: 192.168.2.2, Local port: 179
  Foreign host: 192.168.3.3, Foreign port: 44312
```

**同一个症状的其他两个变体**

| 变体 | 证据 | 修正 |
|------|------|------|
| 两侧都配了 `update-source`，但用的是**不同接口**（一边 Lo0、一边 Lo1） | `show tcp brief` 里源地址不是对端配置的邻居地址 | 统一用同一个 Lo 接口 |
| EBGP 用 Lo0 建邻，漏了 `ebgp-multihop` | `Connect` → `Active` 循环，`ping Lo0` 通 | 加 `ebgp-multihop 3`（XR/NX-OS 在 AF 下） |

> **一句话记法**：IBGP 走 Lo0 = **路由 + update-source**（两条）；EBGP 走 Lo0 = **路由 + update-source + ebgp-multihop**（三条）。

---

### 用例 4：router-id 冲突——每 30 秒掉一次的"幽灵抖动"

**症状**：R3 与 R4 的 IBGP 每隔 20～30 秒断一次，恢复后又能通，日志成对出现 `Down`/`Up`，业务表现为"间歇性丢包"。

**步骤 1 — 抓 `Last reset` 与 NOTIFICATION**

```
! R3 (NX-OS)
show bgp ipv4 unicast neighbors 192.168.4.4 | include "Last reset|Notification|state"
  BGP state = Idle
  Last reset 00:00:04, due to BGP Notification sent
  Notification sent: 2/3 (BGP error code 2, subcode 3)
```

**步骤 2 — 2/3 = Bad BGP Identifier，直接对比双方 router-id**

```
show bgp ipv4 unicast neighbors 192.168.4.4 | include "router ID"
  Local router ID: 192.168.3.3
  Remote router ID: 192.168.3.3        ← 完全一样，这就是根因
```

```
! 对端 R4 (IOS) 视角同样能看到
show bgp ipv4 unicast neighbors 192.168.3.3 | include "router ID"
  Local router ID: 192.168.3.3
  Remote router ID: 192.168.3.3
```

**根因**：两台设备从同一个模板克隆，Lo0 地址与 `router-id` 都没改。BGP 用 router-id 作为设备身份做防环，相同即视为"自己和自己在说话"。

**步骤 3 — 修正 router-id（三平台）**

```
! R4 (IOS)
router bgp 65200
 bgp router-id 192.168.4.4
```

```
! R3 (NX-OS)：注意 NX-OS 的 router-id 不是 bgp 前缀
router bgp 65200
  router-id 192.168.3.3
```

```
! R2 (IOS XR)
router bgp 65200
 bgp router-id 192.168.2.2
```

**步骤 4 — 验证冲突已解除**

```
! 改完 router-id 会自动重置所有邻居，无需 clear
show bgp ipv4 unicast neighbors 192.168.4.4 | include "router ID|state|reset"
  BGP state = Established, up for 00:01:12
  Local router ID: 192.168.3.3
  Remote router ID: 192.168.4.4
  Last reset 00:01:12, due to BGP Notification sent
  Connections established 14; dropped 13        ← 数字停止增长即为修复
```

**如何提前发现（批量设备必做）**

```
! 一条命令扫全网 router-id，找重复
show bgp ipv4 unicast summary | include "routerId|Router ID"
BGP router identifier 192.168.3.3, local AS number 65200
! XR
show bgp ipv4 unicast summary | include "routerId"
BGP router identifier 192.168.2.2, local AS number 65200
```

| 检查项 | 说明 |
|--------|------|
| `router-id` 全网唯一 | 与 Lo0 解耦，用 `10.<region>.<site>.<id>` 规划 |
| 克隆配置后必改 | 主机名、Lo0、router-id 三件套 |
| 改 router-id 的影响 | **重置该实例下所有邻居**，需窗口 |
| VRF 内独立 | 不同 VRF 的 BGP 实例 router-id 可以相同，不冲突 |

---

### 用例 5：MD5 认证——"配了一边"和"密文搬运"

**症状**：R1–R2 会话在 `Active` 与 `Connect` 之间循环。**关键特征：日志里没有任何 BGP NOTIFICATION**，因为连 OPEN 都没换到。

**步骤 1 — 确认不是"正常"的连接失败**

```
! R1 (IOS)
show bgp ipv4 unicast neighbors 10.1.12.2 | include "Last reset|state"
  BGP state = Active
  Last reset 00:00:31, due to TCP connection failed

show tcp brief | include 179
（无 ESTABLISHED）
```

**步骤 2 — 日志里的两类 BADAUTH，含义完全不同**

```
%TCP-6-BADAUTH: No MD5 digest from 10.1.12.2(179) to 10.1.12.1(38876)
```

```
%TCP-6-BADAUTH: Invalid MD5 digest from 10.1.12.2(179) to 10.1.12.1(38876)
```

| 日志 | 含义 | 修哪边 |
|------|------|-------|
| `No MD5 digest` | 对端发来的 TCP option 里**没有** MD5 摘要 | 对端**没配**密码 |
| `Invalid MD5 digest` | 有摘要但校验不过 | 两边都配了，**密码不一致** |

> 排错时**在两台设备上同时看日志**：A 报 `No` 说明 B 没配；A 报 `Invalid` 说明两边字符串不同。方向别搞反。

**步骤 3 — 三平台核对密码（注意"看起来一样"的密文陷阱）**

```
! IOS：默认存储为 type 7
show running-config | include "neighbor 10.1.12.2 password"
 neighbor 10.1.12.2 password 7 0822455D0A16544541
```

```
! XR：写的是 clear 还是 encrypted
show running-config router bgp 65200 neighbor 10.1.12.1
 neighbor 10.1.12.1
  remote-as 65100
  password clear Cisco123
```

```
! NX-OS：password 3 表示 3DES 加密存储
show running-config bgp | include "password"
    password 3 7e1d5b1a0c9f2d3e4a5b6c7d8e9f0a1b
```

**"看起来完全一样却不通"的三种真实情况**

| 情况 | 为什么会发生 | 判据 |
|------|-------------|------|
| 密文从旧设备直接复制到新设备 | type 7 / 3DES 密文**绑定设备密钥**，不可跨设备移植 | 两台都改成明文重配 |
| 一侧 `password clear`、另一侧 `password 3` 混用不同算法 | 中间版本差异，摘要算法不一致 | 统一配置方式，改完 `clear` 一次 |
| 中间有 LB / TCP 代理 | 设备改写或剥离 TCP option（MD5 在 TCP option 里） | 旁路设备，直连测试 |

**步骤 4 — 统一配置（三平台标准写法）**

```
! IOS
router bgp 65100
 neighbor 10.1.12.2 password Cisco123
```

```
! XR
router bgp 65200
 neighbor 10.1.12.1
  password clear Cisco123
```

```
! NX-OS
router bgp 65300
  neighbor 10.1.23.2
    password 0 Cisco123        ! 0 = 明文存储，便于比对；生产可用 3
```

**步骤 5 — 验证**

```
show bgp ipv4 unicast summary
10.1.12.2       4 65200       6       7        1    0    0 00:00:14          1

show logging | include BADAUTH
（不再出现）
```

> **演进方向**：MD5 存在已知弱点且不支持密钥轮换，XR/NX-OS 已支持 **TCP-AO**（HMAC-SHA-1-96）。但 IOS 支持受限，跨平台部署前必须查各版本 release notes——**协议支持不对等时，一侧启用就会直接断**。

---

### 用例 6：`Established` 但 `PfxRcd` 永远是 0

**症状**：会话稳定，`Up/Down` 几个小时，`PfxRcd` 一直是 0。日志干干净净——**没有错，也没有路由**。

**步骤 1 — 先判断"没生成"还是"没发出"还是"没接收"**

| 发送端 `advertised-routes` | 接收端 `PfxRcd` | 结论 |
|---------------------------|----------------|------|
| 空 | 0 | **发送端问题**：`network` 未生效 / XR 缺 `route-policy out` / AF 未激活 |
| 有 | 0 | **接收端问题**：AF 未 `activate` / 入向策略全拒 / AS_PATH 环路 / RT 不匹配 |

```
! 发送端 R2 (XR)
show bgp ipv4 unicast neighbors 10.1.12.1 advertised-routes
% No prefixes advertised to any neighbor     ← 空，问题在发送端
```

**步骤 2 — 发送端三种根因**

**(a) XR 忘了 `route-policy out`**（跨厂商第一大坑）

```
! 错：有 AF，没有出向策略
neighbor 10.1.12.1
 address-family ipv4 unicast
  route-policy PASS-ALL in          ← 只配了 in
```

```
! 对：in / out 都要显式关联
neighbor 10.1.12.1
 address-family ipv4 unicast
  route-policy PASS-ALL in
  route-policy PASS-ALL out
```

**(b) NX-OS 忘了在邻居下进 AF**

```
! 错：只在全局 AF 里写了 network
router bgp 65300
  address-family ipv4 unicast
    network 172.16.3.0/24
  neighbor 10.1.23.2
    remote-as 65200
```

```
! 对：邻居下必须有 AF 段
router bgp 65300
  address-family ipv4 unicast
    network 172.16.3.0/24
  neighbor 10.1.23.2
    remote-as 65200
    address-family ipv4 unicast
```

**(c) IOS 关了默认激活却忘了 `activate`**

```
! 错
router bgp 65100
 no bgp default ipv4-unicast
 neighbor 10.1.12.2 remote-as 65200

! 对
router bgp 65100
 no bgp default ipv4-unicast
 neighbor 10.1.12.2 remote-as 65200
 address-family ipv4
  neighbor 10.1.12.2 activate
```

**步骤 3 — 接收端的判据：看能力行有没有 `received`**

```
show bgp ipv4 unicast neighbors 10.1.12.2 | include "Address family|BGP state"
  BGP state = Established, up for 03:12:44
  Address family IPv4 Unicast: advertised and received     ← 双向都协商成功
```

| 输出 | 含义 |
|------|------|
| `advertised and received` | AF 协商正常，问题在策略或前缀生成 |
| 只有 `advertised` | 对端没在本端这个 AF 上发路由（对端 AF 未激活） |
| 整行缺失 | 双方 AF 不一致（如一端配了 `ipv4 multicast`），根本没协商 |

**步骤 4 — 策略层拒绝的证据（软重置后看计数器）**

```
! IOS
clear ip bgp 10.1.12.2 soft in
show bgp ipv4 unicast neighbors 10.1.12.2 | include "Prefix activity|accepted|filtered"
  Prefix activity:               ----       ----
    Prefixes Current:               0          0
    Prefixes Total:                 0          0
  Local Policy Denied Prefixes:    -----      -----
    inbound route-map:                 0          0
    inbound prefix-list:               0          0
```

```
! XR
clear bgp ipv4 unicast 10.1.12.1 soft in
show bgp ipv4 unicast neighbors 10.1.12.1 | include "Prefix|Rejected|policy"
  Prefixes Rejected by inbound policy: 0
```

> **反向思路**：如果接收端策略计数器全是 0，而发送端 `advertised-routes` 也是空，那就回到第 1 章的 `network` 规则——**前缀根本没进 Loc-RIB**，跟策略无关。这条判据能省下一半时间。

---

### 用例 7：每 90 秒抖一次——单向丢包与 max-prefix

#### 场景 A：`hold time expired`，KEEPALIVE 只发不收

**症状**：会话每隔约 90 秒掉一次，业务间歇性中断，`show logging` 时间戳非常规律。

**步骤 1 — 量化抖动**

```
! R1 (IOS)
show bgp ipv4 unicast neighbors 10.1.12.2 | include "Last reset|established|dropped|BGP state"
  BGP state = Active
  Last reset 00:00:12, due to hold timer expired
  Connections established 41; dropped 40
```

**步骤 2 — 看收发是否对称（这一步直接指向根因）**

```
! 输出节选，只看 Message statistics 段
show bgp ipv4 unicast neighbors 10.1.12.2
  Message statistics:
                         Sent       Rcvd
    Opens:                  5          5
    Notifications:          0          0
    Updates:                3          2
    Keepalives:           812          6      ← 发了 812 个，只收到 6 个
```

**判据**：`Sent` 远大于 `Rcvd`，而 OPEN/UPDATE 都正常交换过，说明**去程通、回程丢**——不是本机不发，是对端的 KEEPALIVE 回不来。

**步骤 3 — 三种典型回程丢失**

| 根因 | 证据 | 修正 |
|------|------|------|
| 中间防火墙 idle timeout < KEEPALIVE 间隔（如 30s vs 60s） | 防火墙会话表里 179 会话反复新建；间隔恰等于 timeout | 防火墙 timeout 调到 ≥ 300s，或把 `timers 20 60` |
| 回程路由不对称 + uRPF 严格模式 | 对端 `show ip cef`/`show route` 回程指向另一条路；本机 uRPF 丢弃计数增长 | 修路由对称，或改 `ip verify unicast source reachable-via any` |
| 中间设备 ACL 只放行了 SYN 方向 | 无 BGP 层错误，纯粹丢 KEEPALIVE | 两侧双向放行 179 |

**步骤 4 — 用计时器规避（三平台）**

```
! IOS
neighbor 10.1.12.2 timers 20 60
! XR
neighbor 10.1.12.1
 timers 20 60
! NX-OS
neighbor 10.1.23.2
  timers 20 60
```

> KEEPALIVE 从 60s 缩到 20s，能"跑赢"30s 的防火墙老化。但这只是绕过，**根因仍在防火墙**，长期应改设备 timeout。

**步骤 5 — 验证修复**

```
show bgp ipv4 unicast neighbors 10.1.12.2 | include "Keepalives|dropped|BGP state"
  BGP state = Established, up for 02:41:07
    Keepalives:          483          481      ← 收发基本对称
  Connections established 42; dropped 40        ← 停止增长
```

#### 场景 B：`max-prefix` 把整个会话踢掉

**症状**：某天上午业务大面积中断，会话全断，日志出现 `MAXPFX`，但带宽、CPU 都正常。

**步骤 1 — 认出 6/1**

```
%BGP-3-MAXPFX: Number of prefixes received from 10.1.12.2 (10000) reaches maximum 10000 for peer 10.1.12.2
%BGP-5-ADJCHANGE: neighbor 10.1.12.2 Down BGP Notification sent
```

```
show bgp ipv4 unicast neighbors 10.1.12.2 | include "Notification|Maximum"
  Notification sent: 6/1 (Cease/Maximum Number of Prefixes Reached)
  Maximum prefixes allowed 10000
  Threshold for warning message 75%, restart interval 5 min
```

**步骤 2 — 找泄漏源：不是"多收了几条"，是"该收的没筛"**

```
! 接收侧看收到的前缀分布（按掩码长度粗看）
show bgp ipv4 unicast neighbors 10.1.12.2 received-routes | include "/2[0-4]"
! 大量 /24 从对端涌进来，说明对端做了 redistribute 而未汇总
```

```
! 发送端确认是否有意外泄漏
show bgp ipv4 unicast neighbors 10.1.12.1 advertised-routes
! 若出现 172.16.0.0/16 之类的明细洪水，根因在发送端
```

**步骤 3 — 修正：入向过滤 + 提高阈值 + 先告警不断链**

```
! IOS
ip prefix-list PL-IN seq 5 permit 203.0.113.0/24 le 24
router bgp 65100
 address-family ipv4
  neighbor 10.1.12.2 prefix-list PL-IN in
  neighbor 10.1.12.2 maximum-prefix 20000 85 warning-only
```

```
! XR
prefix-set PS-IN
  203.0.113.0/24 le 24
end-set
!
route-policy FILTER-IN
  if destination in PS-IN then
    pass
  else
    drop
  endif
end-policy
!
router bgp 65200
 neighbor 10.1.12.1
  address-family ipv4 unicast
   route-policy FILTER-IN in
   maximum-prefix 20000 85 warning-only
```

```
! NX-OS
ip prefix-list PL-IN seq 5 permit 203.0.113.0/24 le 24
router bgp 65300
  neighbor 10.1.23.2
    address-family ipv4 unicast
      prefix-list PL-IN in
      maximum-prefix 20000 85 warning-only
```

**步骤 4 — 验证**

```
show bgp ipv4 unicast summary
10.1.12.2       4 65200    1842    1790        3    0    0 00:12:41        128
                                                                          ↑ 收敛到合理水位
show bgp ipv4 unicast neighbors 10.1.12.2 | include "Maximum|Threshold"
  Maximum prefixes allowed 20000 (warning-only)
  Threshold for warning message 85%
```

> **`maximum-prefix` 是"熔断器"而不是"过滤器"**：它断的是整个会话，代价是"以一条坏路由换掉全部好路由"。先用 `prefix-list`/`route-policy` 精确过滤，再用 `maximum-prefix ... warning-only` 做兜底监控。

---

## 关键命令速查表

| 命令 | 用途 |
|------|------|
| `show bgp ipv4 unicast summary` | 一眼看状态与 `PfxRcd`，定第①/④段 |
| `show bgp ipv4 unicast neighbors IP` | `Last reset ... due to` + Notification + 能力 + 计时器 |
| `show tcp brief \| include 179` | 179 是否 Established、源地址是否正确 |
| `telnet IP 179` | `refused` = 没监听，`timeout` = 被拦 |
| `show control-plane host open-ports` | 本机是否在监听 179（IOS/NX-OS） |
| `show logging \| include "ADJCHANGE\|NOTIFICATION\|BADAUTH\|MAXPFX"` | 会话历史与关闭原因 |
| `show ip access-lists ACL` / XR `show ipv4 access-lists` | `deny` 计数是否增长 |
| `show bgp ipv4 unicast neighbors IP advertised-routes` | 出向是否真的发了 |
| `clear bgp ipv4 unicast IP soft in` | 改入向策略后重算，不重建 TCP |
| `show tech-support bgp` | 重置前留证，提交 TAC |

---

## 三平台差异速查

| 特性 | IOS | IOS XR | NX-OS |
|------|-----|--------|-------|
| 邻居地址写法 | IP | 可写 `IP/32` | IP |
| `update-source` 位置 | `router bgp` / AF 下 | **`neighbor` 下** | `neighbor` 下 |
| `ebgp-multihop` 位置 | `router bgp` / AF 下 | **AF 下** | AF 下 |
| `ttl-security` | `neighbor x ttl-security hops n` | AF 下 `ttl-security` | AF 下 `ttl-security hops n` |
| 计时器 | `neighbor x timers ka hold` | `neighbor` 下 `timers ka hold` | `neighbor` 下 `timers ka hold` |
| 认证 | `neighbor x password` | `password clear/encrypted` | `password 0/3` |
| `maximum-prefix` | `neighbor x maximum-prefix` | AF 下 | AF 下 |
| BFD | `neighbor x fall-over bfd` | `bfd fast-detect` | AF 下 `bfd` |
| 被动模式 | `transport connection-mode passive` | 同左 | 同左 |
| 日志 | `show logging` | `show logging` | `show logging log` |
| 默认策略 | 全放行 | **必须关联 `route-policy`** | 需显式 AF |

---

## 本章核心要点

1. **对等体建立分四段**：TCP → OPEN → KEEPALIVE → AF 协商，`Established` 只代表第三段通过
2. **`St/PfxRcd` 是数字还是文字**，是 30 秒内定段的最快判据
3. **`telnet <peer> 179`**：`refused` 查对端配置，`timeout` 查中间网络——一分钟分开两类根因
4. **ACL 必须双向放行 179**（含 `src eq bgp`），`deny` 计数不增长就说明问题不在这个 ACL
5. **NOTIFICATION 记住 2/2（AS 错）、2/3（router-id 冲突）、2/6（hold time）、4（hold 超时）、6/1（max-prefix）**
6. **`sent` 与 `received` 决定查哪台设备**：本机发出去的说明本机认为对端死了
7. **`Last reset ... due to` + `Connections established/dropped`** 是抖动排错的两个必看字段
8. **收发不对称（Keepalives 812/6）= 单向丢包**，优先查防火墙 idle timeout 与非对称路由
9. **IBGP 走 Lo0 要两条（路由 + update-source），EBGP 走 Lo0 要三条（多 `ebgp-multihop`）**
10. **XR 不关联 `route-policy` 就静默丢弃/不发送**，跨厂商对接第一坑；NX-OS 邻居下必须显式进 AF
11. **router-id 全网唯一**，克隆配置后必改；改 router-id 会重置该实例全部邻居
12. **`maximum-prefix` 是熔断器不是过滤器**，先做前缀过滤，再用 `warning-only` 兜底

---

[← 第 2 章 · 通用的排错方法](chapter-02-troubleshooting-methodology.md) | [← 返回首页](../) | 第 4 章 · 路由通告和 BGP 策略（待精读）
