# 第 1 章 · BGP 基础

> **本章涵盖**：BGP 消息和路由器间通信；IOS、IOS XR 和 NX-OS 中的 BGP 基本配置；IBGP 规则；EBGP 规则；BGP 路由聚合。

---

## 1.1 边界网关协议（BGP）

RFC 1654 将 BGP 定义为符合 EGP 标准的 **路径矢量路由协议**，提供了可扩展性、灵活性和网络稳定性。BGP 是唯一用来在 Internet 上交换网络信息的路由协议，维护着超过 600,000 条 IPv4 路由。

**BGP 与 IGP 的本质区别**：
- **IGP**（OSPF/IS-IS）：在同一组织内寻找**最短路径**，通过增量更新维护拓扑
- **BGP**（EGP）：在组织之间交换路由，每个组织可实施**独立路由策略**，优先考虑**稳定性**而非最短路径

BGP 不像 OSPF/IS-IS 那样通告增量更新，也不周期性刷新通告。一条链路的翻动可能导致上千条路由重算，因此 BGP 设计上刻意保守。

### 1.1.1 自治系统号（ASN）

| 属性 | 2 字节 ASN（旧） | 4 字节 ASN（RFC 4893） |
|------|-----------------|----------------------|
| 总量 | 65,535 | 4,294,967,295 |
| 私有范围 | 64,512 ~ 65,535 | 4,200,000,000 ~ 4,294,967,294 |

IANA 负责分配公共 ASN。申请公共 ASN 需要：
- 拥有公共分配的 IP 地址段
- 需要多条 Internet 连接
- 需要独立的路由策略

> ⚠️ 私自使用其他组织的 ASN 会导致流量丢失并对 Internet 造成严重影响。

### 1.1.2 路径属性（PA）

BGP 在每个前缀上附加路径属性，用于粒度化路由策略控制。PA 分为四类：

| 类别 | 说明 |
|------|------|
| **公认必遵** | 所有 BGP 设备必须识别，必须包含在每个前缀通告中 |
| **公认自决** | 所有 BGP 设备必须识别，可包含也可不包含 |
| **可选传递** | 不要求所有设备识别，可跨 AS 传递 |
| **可选非传递** | 不要求所有设备识别，不可跨 AS 传递 |

**NLRI（网络层可达性信息）** = 前缀 + 前缀长度 + BGP PA

### 1.1.3 环路预防

BGP 作为路径矢量协议，不像链路状态协议那样拥有完整拓扑。环路预防的核心机制是 **AS_PATH** 属性（公认必遵）：

- AS_PATH 列出前缀从源 AS 开始穿越的所有 ASN
- 如果路由器在前缀的 AS_PATH 中看到自己的 ASN，就丢弃该前缀

### 1.1.4 地址家族（AFI/SAFI）

RFC 2858 引入多协议 BGP（MP-BGP），通过 AFI + SAFI 扩展到多种协议：

| AFI | SAFI | 网络层信息 |
|-----|------|-----------|
| 1 | 1 | IPv4 单播 |
| 1 | 2 | IPv4 组播 |
| 1 | 4 | 携带 MPLS 标签的 IPv4 单播 |
| 1 | 128 | MPLS L3VPN IPv4 |
| 2 | 1 | IPv6 单播 |
| 2 | 4 | 携带 MPLS 标签的 IPv6 单播 |
| 2 | 128 | MPLS L3VPN IPv6 |
| 25 | 65 | VPLS / VPWS |
| 25 | 70 | EVPN |

每个地址家族维护独立的数据库和路由策略。MP-BGP 通过 `MP_REACH_NLRI` 和 `MP_UNREACH_NLRI` 属性承载不同地址家族的 NLRI。

### 1.1.5 BGP 会话类型

| 类型 | 说明 | 管理距离（AD） |
|------|------|---------------|
| **IBGP** | 同一 AS 内的 BGP 会话 | 200 |
| **EBGP** | 不同 AS 间的 BGP 会话 | 20 |

---

## 1.2 路由器间通信

**BGP 的核心设计选择：使用 TCP 179 端口进行通信，而非自定义 Hello 包。**

- TCP 提供分片、排序、确认和重传
- BGP 不自动发现邻居 — 邻居通过 IP 地址静态定义
- BGP 可建立直连会话，也可建立**多跳会话**（跨越网络边缘）
- 多跳会话需要 RIB 中的底层路由（静态或 IGP），**默认路由不足以建立多跳 BGP 会话**

这使得 BGP 可以脱离数据平面 — 控制平面路由器不必在数据转发路径中。

---

## 1.3 BGP 消息类型

| 类型 | 名称 | 功能 |
|------|------|------|
| 1 | OPEN | 建立邻接关系，协商会话能力 |
| 2 | UPDATE | 通告、更新或撤销路由 |
| 3 | NOTIFICATION | 告知错误条件，导致会话关闭 |
| 4 | KEEPALIVE | 确保邻居在线（心跳） |

### OPEN 消息

包含：BGP 版本、ASN、保持时间、BGP 识别符（RID）、可选参数。

- **保持时间**：双方协商使用较小值，最小 3 秒或为 0。Cisco 默认 180 秒。
- **BGP RID**：32 位唯一标识，手动设置是最佳实践。

RID 动态分配逻辑差异：

| 平台 | 逻辑 |
|------|------|
| IOS | 最大 up 环回 IP → 最大 up 接口 IP |
| IOS XR | 最小 up 环回 IP → 0（阻止 BGP 会话建立） |
| NX-OS | 最小 up 环回 IP → 最小 up 接口 IP |

### KEEPALIVE

- 间隔 = 保持时间 / 3（Cisco 默认 60 秒）
- 保持时间为 0 时不交换 KEEPALIVE

### UPDATE 消息

- 携带 NLRI（前缀 + PA）用于通告
- 携带仅含前缀的撤销 NLRI
- 可兼作 KEEPALIVE 以减少不必要的流量

### NOTIFICATION 消息

检测到错误时发送（保持计时器超时、能力变更、重置请求），发送后连接关闭。

---

## 1.4 BGP 邻居状态（有限状态机）

BGP 会话建立经历 6 个状态：

```
Idle → Connect → Active → OpenSent → OpenConfirm → Established
```

| 状态 | 说明 |
|------|------|
| **Idle** | 初始状态。启动 TCP 连接并监听。失败后 ConnectRetry 计时器 = 60 秒，之后每次翻倍 |
| **Connect** | 初始化 TCP 连接。TCP 三次握手成功 → 发送 OPEN → 进入 OpenSent。计时器超时 → 进入 Active |
| **Active** | 尝试新的 TCP 三次握手。成功 → 发送 OPEN → 进入 OpenSent。失败 → 回到 Connect |
| **OpenSent** | 已发送 OPEN，等待对方 OPEN。检查：版本匹配、源 IP 匹配、AS 号匹配、RID 唯一、安全参数。无错 → 发送 KEEPALIVE → 进入 OpenConfirm |
| **OpenConfirm** | 等待 KEEPALIVE。收到 → 进入 Established。超时/NOTIFICATION → 回到 Idle |
| **Established** | 会话建立成功，通过 UPDATE 交换路由 |

> **排错要点**：BGP 邻居卡在不同状态的常见原因：
> - **Idle**：路由不可达、ACL 阻断、BGP 进程未启动
> - **Active**：TCP 连接失败（端口 179 被防火墙阻断、路由问题）
> - **OpenSent → Idle**：AS 号不匹配、RID 冲突、BGP 版本不匹配
> - **OpenConfirm → Idle**：保持计时器超时

---

## 1.5 BGP 基本配置

### 三平台配置对比

**IOS（默认激活 IPv4）：**

```
router bgp 65100
 neighbor 10.1.12.2 remote-as 65100
```

**IOS（禁用默认激活 — 推荐）：**

```
router bgp 65100
 no bgp default ipv4-unicast
 neighbor 10.1.12.1 remote-as 65100
 !
 address-family ipv4
  neighbor 10.1.12.1 activate
 exit-address-family
```

**IOS XR：**

```
router bgp 65100
 bgp router-id 192.168.1.1
 address-family ipv4 unicast
 !
 neighbor 10.1.12.2
  remote-as 65100
  address-family ipv4 unicast
```

> IOS XR 必须为 EBGP 对等体关联路由策略，否则收到的 NLRI 会被丢弃且不通告任何 NLRI。

**NX-OS：**

```
router bgp 65100
 address-family ipv4 unicast
 neighbor 10.1.12.2 remote-as 65100
  address-family ipv4 unicast
```

### 验证命令

```
show bgp ipv4 unicast summary              ! 查看会话汇总
show bgp ipv4 unicast neighbors 10.1.12.1  ! 查看邻居详细信息
show bgp ipv4 unicast                       ! 查看 BGP 表（Loc-RIB）
```

### BGP 汇总输出关键字段

| 字段 | 说明 |
|------|------|
| Neighbor | 对等体 IP |
| V | BGP 版本 |
| AS | 对等体 AS 号 |
| MsgRcvd/MsgSent | 收发消息数 |
| Up/Down | 会话建立时长（未建立则显示状态） |
| State/PfxRcd | 当前状态或收到的前缀数量 |

### BGP 三张表

| 表 | 作用 |
|----|------|
| **Adj-RIB-In**（入向邻接 RIB） | 原始 NLRI，等待入向策略处理。处理后清除以节省内存 |
| **Loc-RIB**（本地 RIB） | 本地生成 + 从对等体收到的 NLRI。通过有效性检查和下一跳可达性检查后，执行最优路径算法 |
| **Adj-RIB-Out**（出向邻接 RIB） | 出向策略处理后的 NLRI |

### 前缀通告规则

`network` 命令不启用接口上的 BGP，而是将精确匹配的 RIB 前缀放入 Loc-RIB：

| RIB 来源 | 下一跳 | Origin | Weight | MED |
|----------|--------|--------|--------|-----|
| 直连网络 | 0.0.0.0 | i (IGP) | 32,768 | — |
| 静态/路由协议 | RIB 中的下一跳 | i (IGP) | 32,768 | IGP 度量值 |

### BGP 最优路径算法（13 步）

1. **权重**（Weight，Cisco 私有，本地有效）
2. **本地优先级**（LOCAL_PREF，公认自决）
3. **本地源**（network 命令、重分布、汇聚）
4. **AIGP**
5. **最短 AS_PATH**
6. **源类型**（IGP < EGP < Incomplete）
7. **最低 MED**
8. **EBGP 优于 IBGP**
9. **最低 IGP 下一跳度量**
10. **EBGP 最老路由优先**（如果两条都是 EBGP）
11. **最低 RID**
12. **最短集群列表**
13. **最低邻居 IP 地址**

> **BGP 流量工程基本规则**：
> - **修改出向路由策略** → 影响入向流量（别人如何到达你）
> - **修改入向路由策略** → 影响出向流量（你如何到达别人）

---

## 1.6 路由过滤和处理

- **IOS/NX-OS**：filter-list、prefix-list、route-map
- **IOS XR**：route-policy

策略变更后需要重置会话：
- **硬重置**：断开会话，移除路由（最具破坏性）
- **软重置**：使缓存无效，请求对等体重新发送通告

```
clear ip bgp *                    ! 硬重置所有邻居
clear ip bgp 10.1.12.2 soft in    ! 软重置入向
clear ip bgp 10.1.12.2 soft out   ! 软重置出向
```

> 入向软重配置（`soft-reconfiguration inbound`）会保存 Adj-RIB-In 在内存中，消耗大量内存。

---

## 1.7 IBGP

### 为什么 AS 内部需要 BGP？

当一个 AS 为多个 AS 提供传输联通性时，IGP 无法胜任：
- **可扩展性**：Internet 有 60 万+ 路由，IGP 无法承受
- **客户路由**：IGP 用度量值选路，BGP 用路径属性，最优路径不一定最短
- **路径属性**：IGP 无法维护 BGP PA

### IBGP 全互联需求

> **核心规则**：从 IBGP 对等体收到的 NLRI **不能**再通告给其他 IBGP 对等体。
>
> 因为 IBGP 不修改 AS_PATH（AS_PATH 中不会添加本地 ASN），所以没有 AS_PATH 环路检测机制。RFC 4271 规定 AS 内所有 BGP 路由器必须**全互联**。
>
> 会话数公式：n×(n−1)/2。5 台 = 10 个会话，10 台 = 45 个会话。

### 通过环回地址建立对等体（推荐）

优势：
- 链路故障时 BGP 会话不受影响（IGP 通过其他路径找到环回地址）
- 无需重新执行最优路径算法
- 自动支持负载分担（多条等价 IGP 路径）

必须配置 `update-source` 指定源 IP：

```
! IOS
neighbor 10.1.23.3 update-source loopback0

! IOS XR / NX-OS（在邻居配置下）
update-source loopback0
```

---

## 1.8 EBGP

### EBGP 与 IBGP 的行为差异

| 特性 | EBGP | IBGP |
|------|------|------|
| TTL | 1 | 255 |
| 下一跳 | 修改为自己的 IP | 不修改 |
| AS_PATH | 添加自己的 ASN | 不添加 |
| 环路检查 | 检查 AS_PATH 中有无本地 ASN | 无 AS_PATH 检查 |

### 下一跳可达性问题

> **经典排错场景**：EBGP 前缀通过 IBGP 传递后下一跳不可达。
>
> R4 → R3（EBGP）→ R2（IBGP）：R3 通告给 R2 时**不修改下一跳**（保持 10.1.34.4）。R2 没有 10.1.34.4 的路由 → 前缀标记为 inaccessible → 不执行最优路径算法 → 不通告给其他对等体。
>
> **解决方案**：
> 1. 将直连链路网络通告到 IGP 或 BGP
> 2. 使用 `next-hop-self` 修改下一跳

### next-hop-self

```
! IOS
neighbor 10.1.23.3 next-hop-self           ! 仅修改 EBGP 前缀的下一跳
neighbor 10.1.23.3 next-hop-self all       ! 也修改 IBGP 前缀的下一跳

! IOS XR / NX-OS
next-hop-self
```

---

## 1.9-1.10 IBGP 扩展性与路由反射器

### 路由反射器（RR）规则

| 规则 | RR 从…收到 NLRI | 通告给 |
|------|----------------|--------|
| 规则 1 | 非 RR 客户端 | 仅 RR 客户端 |
| 规则 2 | RR 客户端 | RR 客户端 + 非 RR 客户端 |
| 规则 3 | EBGP 对等体 | RR 客户端 + 非 RR 客户端 |

### RR 环路预防属性

| 属性 | 类型 | 作用 |
|------|------|------|
| **ORIGINATOR_ID** | 可选非传递 | 第一台 RR 创建，值为注入路由的路由器 RID。收到以自己 RID 为 Originator 的 NLRI 时丢弃 |
| **CLUSTER_LIST** | 非传递 | RR 附加自己的集群 ID。收到包含自己集群 ID 的 NLRI 时丢弃 |

```
! 配置 RR 客户端
! IOS
neighbor 10.1.34.4 route-reflector-client

! IOS XR / NX-OS（邻居地址家族下）
route-reflector-client

! 设置集群 ID
bgp cluster-id 192.168.2.2
```

### 带外路由反射器

专门的 RR 路由器，不在数据路径中：
- 只占用 BGP 路由表内存和处理能力
- **不应使用 next-hop-self**（会把 RR 放到数据路径中）
- MPLS L2VPN/L3VPN 中常用多个带外 RR

### BGP 联盟（Confederation）

将一个大 AS 分为多个子 AS（成员 AS）：

- 成员 AS 使用私有 ASN（64512~65535）
- 联盟外部对等体不知道内部子 AS 结构
- AS_PATH 中用 `AS_CONFED_SEQUENCE`（括号显示）防止环路，但不计入 AS_PATH 长度
- MED 和 LOCAL_PREF 可在成员 AS 间传递，但不离开联盟
- 路由离开联盟时移除 AS_CONFED_SEQUENCE

```
router bgp 65100                           ! 成员 AS
 bgp confederation identifier 200           ! 联盟标识符
 bgp confederation peers 65200              ! 对等成员 AS
```

---

## 1.12 BGP 团体（Community）

- 可选传递属性，32 位号码
- 新格式：`AA:NN`（0~65535:0~65535）
- RFC 4360 扩展为扩展团体属性（用于 VPN）

| 平台 | 默认行为 |
|------|---------|
| IOS | 默认不通告给对等体，需 `send-community` |
| NX-OS | 默认不通告，需 `send-community` |
| IOS XR | 默认通告给 IBGP，EBGP 需 `send-community-ebgp` |

---

## 1.13 路由汇总

### 两种汇总技术

| 技术 | 说明 | 缺点 |
|------|------|------|
| 静态 | Null0 静态路由 + `network` 通告 | 网络不可达仍通告 |
| 动态 | `aggregate-address` 自动创建聚合路由 | 需要至少一条明细路由存在 |

### 聚合命令

```
! IOS
aggregate-address 172.16.0.0 255.255.252.0 [summary-only] [as-set] [suppress-map] [advertise-map]

! IOS XR
aggregate-address 172.16.0.0/22 [summary-only] [route-policy] [as-set] [advertise-map]

! NX-OS
aggregate-address 172.16.0.0/22 [summary-only] [suppress-map] [as-set] [advertise-map]
```

### 原子聚合（Atomic Aggregate）

- 聚合路由丢失原始 AS_PATH、MED、Community 等属性
- 标记为 `atomic-aggregate` 表示路径信息不完整
- 聚合路由的 AS_PATH 只显示聚合路由器的 ASN

### AS_SET

使用 `as-set` 关键字保留原始前缀的 BGP 属性历史：
- AS_PATH 中的 AS_SET 显示在括号中（即使多个 AS 也只计为 1 跳）
- 可与 `advertise-map` 配合，有选择地复制属性

### 默认路由通告

```
default-information originate              ! 通告默认路由（RIB 中必须存在）
neighbor 10.1.12.2 default-originate       ! 向指定邻居通告默认路由（不需要 RIB 中存在）
```

---

## 1.14-1.16 私有 AS 处理

### 移除私有 AS

```
neighbor 10.1.12.2 remove-private-as
```

规则：
- 仅移除通告给 EBGP 对等体的路由中的私有 ASN
- AS_PATH 中只有私有 ASN 时才移除
- 同时有私有和公共 ASN 时不移除

### 允许 AS（Allow AS）

允许路由器在 AS_PATH 中看到自己 ASN 时仍接收该路由（用于传输 AS 连接两个相同 AS 的站点）：

```
neighbor 10.1.12.2 allowas-in
```

### 本地 AS（Local AS）

网络融合时保留旧 ASN：

```
! IOS
neighbor 10.1.12.2 local-as 65100 [no-prepend [replace-as [dual-as]]]
```

| 关键字 | 行为 |
|--------|------|
| （默认） | 替换 ASN 添加到收发路由的 AS_PATH 中 |
| `no-prepend` | 接收路由时不附加替换 ASN |
| `no-prepend replace-as` | 收发都不附加，完全模拟原始 AS |
| `dual-as` | 允许对等体使用任一 ASN 建立会话 |

---

## 关键命令速查表

| 命令 | 用途 |
|------|------|
| `show bgp ipv4 unicast summary` | 查看所有 BGP 邻居汇总 |
| `show bgp ipv4 unicast neighbors IP` | 查看指定邻居详细信息 |
| `show bgp ipv4 unicast` | 查看 BGP 表（Loc-RIB） |
| `show bgp ipv4 unicast NETWORK` | 查看指定前缀的 BGP 属性 |
| `show tcp brief` | 查看 TCP 连接状态 |
| `show bgp ipv4 unicast update-group` | 查看更新组 |
| `clear ip bgp *` | 硬重置所有 BGP 邻居 |
| `clear ip bgp IP soft in/out` | 软重置指定邻居 |

---

## 三平台差异速查

| 特性 | IOS | IOS XR | NX-OS |
|------|-----|--------|-------|
| IPv4 默认激活 | 是 | 否 | 否 |
| EBGP 需路由策略 | 否 | 是 | 否 |
| RID 选择 | 最大环回 IP | 最小环回 IP | 最小环回 IP |
| RID 无环回时 | 最大接口 IP | 0（阻止会话） | 最小接口 IP |
| 路由过滤 | prefix-list/route-map | route-policy | prefix-list/route-map |
| 团体属性默认通告 | 否 | IBGP 通告 | 否 |
| 外部路由标记 | 无 | 无 | e（外部）/ l（本地） |

---

## 本章核心要点

1. **BGP = 路径矢量协议**，核心是 AS_PATH 环路预防和路径属性驱动的策略路由
2. **TCP 179 端口**，支持多跳会话，控制平面可脱离数据平面
3. **6 状态有限状态机**：Idle → Connect → Active → OpenSent → OpenConfirm → Established
4. **IBGP 全互联规则**：从 IBGP 学到的路由不转发给其他 IBGP 对等体
5. **三张表**：Adj-RIB-In → Loc-RIB → Adj-RIB-Out
6. **13 步最优路径算法**：Weight → LOCAL_PREF → 本地源 → AIGP → AS_PATH → Origin → MED → EBGP>IBGP → IGP 下一跳 → 最老 → RID → 集群列表 → 邻居 IP
7. **扩展性方案**：路由反射器（RR + ORIGINATOR_ID + CLUSTER_LIST）、联盟
8. **下一跳可达性**是最常见的 IBGP 排错问题 — 用 `next-hop-self` 解决

---

[← 返回首页](../) | [第 2 章 · 通用的排错方法 →](chapter-02-troubleshooting-methodology.md)
