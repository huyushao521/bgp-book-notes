# 第 2 章 · 通用的排错方法

> **本章涵盖**：结构化排错的四个固定动作；六大通用排错方法及其适用场景；如何判定控制平面还是数据平面故障；基线与变更管理；IOS / IOS XR / NX-OS 通用命令对照；ping、traceroute、debug、日志、抓包等工具的陷阱；向 TAC 求助前的信息准备；**四个三平台（IOS / IOS XR / NX-OS）实战用例**。

---

## 2.1 为什么需要方法论

同一张故障单，老手十分钟定位，新手折腾两小时——差别通常不在命令记得多少，而在**"下一步做什么"的顺序是自洽的**。

**随性排错（shoot from the hip）的三个代价**：

1. **在错误方向上消耗时间**——改配置、清邻居、重启单板，问题依旧
2. **掩盖真实根因**——规避措施把现象消掉了，几周后换个形式爆发
3. **引入二次故障**——排错动作本身造成新的中断

方法论的价值：把"猜"变成"验证"。每步可复现、可回滚、可解释，哪怕最终没解决，也留下有效线索。

> ⚠️ 生产铁律：**一次只改一个变量**。一次改三处，即使恢复也说不清哪处生效，下次同样故障你依然不会。

### 2.1.1 排错的四个固定动作

| 阶段 | 目标 | 产出 |
|------|------|------|
| 1. 定义问题 | 说清现象、范围、时间点、影响面 | 一句话故障描述 |
| 2. 收敛范围 | 缩小到具体设备 / 协议 / 方向 | 故障树上的一个分支 |
| 3. 验证假设 | 用命令证明或推翻 | 关键 show 输出对比 |
| 4. 修复与回归 | 最小改动修复并记录 | 变更记录 + 复盘结论 |

### 2.1.2 定义问题：先问清五个问题

| 问题 | 为什么必须问 | 示例 |
|------|-------------|------|
| **什么不通** | 区分完全不通 / 部分不通 / 时通时不通 / 慢 | 只有 `10.1.1.0/24` 不通 |
| **从哪到哪** | 明确方向，单向故障最易误判 | A 能 ping 通 B，B ping 不回 A |
| **什么时候开始** | 对齐变更、告警、日志时间轴 | 昨晚 22:00 割接之后 |
| **影响多大** | 决定是否走应急通道 | 单条前缀 / 整个 POP 出口 |
| **之前正常吗** | 区分"从没通过"与"曾经通过" | 新链路从未通过 → 配置问题 |

> "**从没通过**"和"**曾经通过**"是两类完全不同的故障：前者多为配置/设计错误，后者多为变更、硬件、资源耗尽。这一问能省一半工作量。

---

## 2.2 六大通用排错方法

### 2.2.1 自底向上（Bottom-Up）

物理 → 数据链路 → IP 可达 → 路由协议 → 应用。

- **适用**：新部署、链路刚起、怀疑光模块/硬件
- **优点**：不漏底层问题，责任边界清晰
- **缺点**：BGP 这类控制面问题要爬到第 4 层才发现，效率低

### 2.2.2 自顶向下（Top-Down）

从业务现象出发逐层向下拆解。

- **适用**：业务侧已给出明确现象或报错
- **优点**：直接命中痛点，优先恢复业务
- **缺点**：底层硬件问题容易被拖到最后

### 2.2.3 分而治之（Divide and Conquer）

把路径或协议栈从中间切开，先判断问题在哪一半，再继续二分。

- **适用**：跨多跳、跨多 AS 的长路径故障
- **做法**：在路径中点执行 `ping` / `traceroute` / `show bgp`，判断"到这里是否正常"，再对异常的那一半二分

> 每二分一次省一半范围，十跳路径最多四次即可收敛到单跳。

### 2.2.4 沿路径追踪（Follow the Path）

按数据流向逐跳对齐"路由表 → 转发表 → 邻居表"。

- **适用**：路由存在但转发不通
- **做法**：路径上**每一跳执行同一组命令**，找第一处出现差异的点

| 每跳必看 | 目的 |
|----------|------|
| `show ip route <prefix>` | 路由是否存在、下一跳是谁 |
| `show ip cef <prefix> detail` | 是否已装 FIB、邻接是否完整 |
| 接口计数 / MTU | 是否有丢弃、MTU 是否不一致 |

### 2.2.5 配置比较（Compare Configurations）

找一台**同角色、工作正常**的设备做对照，逐段 diff。

- **适用**：多台同类设备中只有一台异常
- **价值**：BGP 排错最有效的招数之一，一眼看出漏配的 `next-hop-self`、`route-policy`、`activate`、`send-community`
- **注意**：比较**语义**而非文本——三平台等价配置长得完全不同

### 2.2.6 组件替换（Component Swap）

用已知正常的部件替换可疑部件（光模块、板卡、链路、整机）。

- **适用**：怀疑硬件、光层、单板转发异常
- **前提**：有冗余路径或已进窗口
- **风险**：替换本身可能引起二次中断，先确认不会扩大影响面

### 方法对比

| 方法 | 最适用场景 | 主要风险 |
|------|-----------|---------|
| 自底向上 | 新上线、物理层故障 | 慢 |
| 自顶向下 | 业务已明确报错 | 漏底层 |
| 分而治之 | 跨多跳、跨 AS 长路径 | 需中间节点权限 |
| 沿路径追踪 | 路由有、转发不通 | 人力成本高 |
| 配置比较 | 同类设备单点异常 | 需参照设备 |
| 组件替换 | 怀疑硬件 | 需窗口、有中断风险 |

> 实战极少单用一种。常见组合：**分而治之**收敛到设备 → **沿路径追踪**定位到跳 → **配置比较**找到具体那几行。

---

## 2.3 先分清：控制平面还是数据平面

> 这是 BGP 排错最关键的一次判断。方向分错，后面所有操作都白做。

| 平面 | 关注点 | 判断命令 |
|------|--------|----------|
| **控制平面** | 邻居能否建立、路由是否学到/通告、策略是否放行 | `show bgp summary` / `neighbors` / `show ip route` |
| **数据平面** | 报文能否真正转发（FIB、邻接、标签栈、MTU、ACL） | `show ip cef` / `ping` / `traceroute` / 接口计数 |

### 2.3.1 三种典型组合

| 路由表 | 实际转发 | 结论 | 排查入口 |
|--------|---------|------|----------|
| 有 | 通 | 正常 | — |
| **有** | **不通** | **数据平面问题** | CEF/FIB、邻接、标签栈、MTU、ACL、uRPF |
| 无 | 不通 | **控制平面问题** | 邻居状态、入/出向策略、下一跳可达性 |

> 反直觉但很常见：**邻居 Established、路由也在表里，业务就是不通**——根因几乎总在转发面（标签没压、MTU 黑洞、ACL 只拦业务流量没拦 ICMP、uRPF 丢非对称回程）。

### 2.3.2 控制面问题的四个检查点

按 BGP 处理顺序，路由"消失"只可能发生在四个位置：

| 位置 | 现象 | 检查 |
|------|------|------|
| 邻居未建立 | summary 里状态不是数字 | `show bgp neighbors` 看最后状态与 Notification |
| 收到但入向被拒 | 邻居 Up，`PfxRcd` 偏小或为 0 | `show bgp neighbors IP` 的 Accepted / 策略计数 |
| 学到但未优选 | 表里有但无 `>` 标记 | `show bgp <prefix>` 看 best 与落选原因 |
| 优选但未通告 | 对端没有该前缀 | 出向策略 + `advertised-routes` |

> `show bgp neighbors IP` 中 **Accepted prefixes** 与 **PfxRcd** 不等，基本可锁定入向策略。

---

## 2.4 基线：排错的一半功夫在故障之前

没有基线，"当前值是否正常"只能靠猜；有基线时，一次 diff 就能看出哪里变了。

### 2.4.1 应长期保存的内容

| 类别 | 内容 | 用途 |
|------|------|------|
| 拓扑 | 物理/逻辑拓扑、AS 号、互联地址、设备角色 | 判断路径与故障域 |
| 邻居清单 | 每台设备应有的 BGP 邻居与 AS | 快速发现"少了一个邻居" |
| 前缀基线 | 每个邻居正常收到的前缀数量 | `PfxRcd` 突降立刻可见 |
| 性能基线 | CPU、内存、flapping 次数、收敛时间 | 判断是否资源型故障 |
| 配置备份 | 每次变更前后的完整配置 | 回滚与 diff |

> `PfxRcd` 是最廉价有效的监控指标：**前缀数量的突变几乎总对应一次策略、链路或上游事件**。

### 2.4.2 变更管理

> 90% 的"莫名其妙"故障，答案就在变更记录里。

变更前必须回答：影响哪些前缀/邻居？回滚怎么做？窗口多长？谁在值守？

变更后必做：核对邻居数量与前缀数、`show bgp summary` 前后对比、关键前缀的 AS_PATH 是否变化。

---

## 2.5 三平台通用命令对照

| 用途 | IOS | IOS XR | NX-OS |
|------|-----|--------|-------|
| BGP 邻居汇总 | `show bgp ipv4 unicast summary` | `show bgp ipv4 unicast summary` | `show bgp ipv4 unicast summary` |
| 邻居明细 | `show bgp ipv4 unicast neighbors 10.1.12.1` | 同 IOS | 同 IOS |
| BGP 表 | `show bgp ipv4 unicast` | `show bgp ipv4 unicast` | `show bgp ipv4 unicast` |
| 指定前缀 | `show bgp ipv4 unicast 10.1.1.0/24` | 同 IOS | 同 IOS |
| 全局路由表 | `show ip route 10.1.1.0` | `show route 10.1.1.0/24` | `show ip route 10.1.1.0` |
| 转发表 | `show ip cef 10.1.1.0 detail` | `show cef 10.1.1.0/24 detail` | `show ip cef 10.1.1.0 detail` |
| 接口状态 | `show ip interface brief` | `show ipv4 interface brief` | `show ip interface brief` |
| 日志 | `show logging` | `show logging` | `show logging log` |
| 一键收集 | `show tech-support` | `show tech-support` | `show tech-support` |
| BGP 专项 | `show tech-support bgp` | `show tech-support bgp` | `show tech-support bgp` |

> **XR 的两个坑**：路由表命令是 `show route`（不是 `show ip route`），且**必须带前缀长度**；接口表是 `show ipv4 interface brief`。

### 2.5.1 出向通告验证（三平台语法一致）

```
show bgp ipv4 unicast neighbors 10.1.12.1 advertised-routes   ! 我通告了什么
show bgp ipv4 unicast neighbors 10.1.12.1 routes              ! 从它收到且被接受的路由
show bgp ipv4 unicast neighbors 10.1.12.1 received-routes     ! 收到但未经入向策略过滤
```

> `received-routes` 需要邻居下配置 `soft-reconfiguration inbound`（若无配置改用 `clear ... soft in` 刷新）。

### 2.5.2 BGP 汇总输出的关键判据

| 列 | 异常信号 | 可能原因 |
|----|---------|---------|
| State/PfxRcd | 显示状态名而非数字 | 会话未建立，看 `neighbors` 输出 |
| State/PfxRcd | 数字远小于预期 | 入向策略、上游过滤、AFI/SAFI 未激活 |
| Up/Down | 时长频繁归零 | flapping：物理层、TCP 重传、MTU、CPU |
| MsgRcvd 增长但 PfxRcd 不增 | 只发 keepalive | 邻居无可用前缀 / 出向策略全拒 |

---

## 2.6 排错工具与陷阱

### 2.6.1 ping：应答由控制面生成

- ping 的 ICMP 应答由**CPU 生成**，受 CoPP / control-plane policing 限速影响：**ping 丢包 ≠ 业务丢包**
- **大包不通、小包通** → 典型 MTU 问题（MPLS 环境多 4~8 字节标签开销）
- 指定源地址，避免回程走错路径：`ping 10.1.1.1 source Loopback0`
- 禁止分片验证 MTU：`ping 10.1.1.1 size 1472 df-bit repeat 5`

### 2.6.2 traceroute：MPLS 下的三个坑

| 现象 | 原因 | 处理 |
|------|------|------|
| 中间跳显示 `* * *` | 该跳未响应 ICMP TTL 超时（策略或限速） | 不代表转发中断，换 `ping` 验证端到端 |
| 完全看不到 P 节点 | MPLS 域内 TTL 传播被抑制 | 用 `ping mpls ipv4` / `traceroute mpls ipv4` 验证 LSP |
| 只显示一条路径 | 基于流的负载分担，只有该五元组实际路径 | 换源/目的端口再测 |

```
ping mpls ipv4 10.1.1.1/32                 ! 验证 LSP 端到端连通性
traceroute mpls ipv4 10.1.1.1/32           ! 逐跳验证标签转发
```

### 2.6.3 debug：能不用就不用

- debug 输出进 CPU，可能直接把控制面打满，**在生产上比故障本身更危险**
- 必须遵守：先 `no logging console` + 开 `logging buffered`，限制时间，用完立即关闭

```
no logging console
logging buffered 100000 debugging
debug ip bgp updates
! ... 复现问题，10~30 秒内关闭 ...
undebug all
show logging | include BGP
```

- XR 用 `debug bgp update` / `debug bgp neighbor-events`；NX-OS 用 `debug bgp updates`，并可用 `logging level bgp` 精细控制
- 条件化 / 精确化：能用 `show` 与 ACL 计数拿到的信息，绝不开 debug

### 2.6.4 日志与告警

| 日志 | 含义 |
|------|------|
| `%BGP-5-ADJCHANGE` | 邻居 Up/Down，含原因（hold timer、administrative、capability） |
| `%BGP-3-NOTIFICATION` | 收到/发出 Notification，错误码直接指向根因 |
| `%BGP-4-MAXPFX` | 达到 maximum-prefix 限制，会话被重置 |
| `%BGP-3-NOMEM` | 内存不足，通常伴随大量前缀抖动 |

- 时间戳统一 UTC 并确保 NTP 同步，否则多设备日志无法对齐
- `logging buffered` 容量要够大（默认值往往只能存几分钟）
- 需要长期趋势时用 SNMP 轮询或 EEM applet 自动采集

### 2.6.5 抓包与流量分析

| 手段 | 适用 | 命令示例 |
|------|------|---------|
| ACL 计数（"穷人探针"） | 确认报文是否到达/丢弃 | `access-list 101 permit tcp host 10.1.12.1 eq 179 any` + `show ip access-lists 101` |
| SPAN / ERSPAN | 交换机侧镜像到分析口 | `monitor session 1 source interface Gi0/1 both` |
| NetFlow / IPFIX | 谁在发、发多少、被谁丢 | `ip flow export ...` |
| NX-OS 内置抓包 | 设备上直接抓控制面流量 | `ethanalyzer local interface inband capture-filter "tcp port 179" limit-captured-frames 200` |
| XR 内置抓包 | 同上 | `monitor capture CAP interface Gi0/0/0/0 both match ipv4 tcp any any` 后 `start` / `show monitor capture CAP buffer brief` |
| IP SLA | 持续探测可用性与延迟抖动 | `ip sla 1` + `icmp-echo 10.1.1.1` |

> BGP 会话问题的终极证据是**抓包看 TCP 179**：三次握手是否完成、是否有 RST、是否卡在 OPEN 之后的 MTU 分片上。

---

## 2.7 向 TAC 求助前该准备什么

工单质量直接决定响应速度。同一个问题，附齐信息的工单可能几小时出结论，只写"BGP 不通"的工单要先来回三轮收集信息。

| 类别 | 具体内容 |
|------|---------|
| 问题描述 | 预期行为 vs 实际行为，一句话说清 |
| 时间线 | 故障开始时间（UTC）、是否与变更重合、当前是否仍存在 |
| 拓扑 | 相关设备、AS 号、互联地址、受影响与正常链路的对比 |
| 设备信息 | 型号、软件版本（`show version`）、角色 |
| 收集包 | 路径上**所有相关设备**的 `show tech-support`（BGP 问题加 `show tech-support bgp`） |
| 命令输出 | `show bgp summary`、`show bgp neighbors IP`、`show bgp <prefix>`、`show logging` |
| 抓包 | 条件允许时提供 TCP 179 抓包（含时间戳） |
| 已做操作 | 改过什么、结果如何、是否已回滚 |

> **不要做的事**：反复 `clear ip bgp *`。每次硬重置都会清空会话与统计，把最能说明问题的证据（flap 历史、Notification、更新组状态）一并清掉。

---

## 2.8 两个高频故障的完整推演

### 2.8.1 邻居 Established 但收不到路由

1. `show bgp ipv4 unicast summary` → 状态是数字，说明会话正常，**问题在路由层不是 TCP 层**
2. `show bgp ipv4 unicast neighbors 10.1.12.1` → 看 `PfxRcd`、`Accepted`、AFI/SAFI 是否激活
3. `clear bgp ipv4 unicast 10.1.12.1 soft in` → 再执行 `received-routes`
   - `received-routes` 有、`routes` 没有 → **入向策略过滤**
   - 两者都空 → 对端压根没发：到对端看 `advertised-routes` 与出向策略
4. 出向策略全拒的常见原因：`route-policy` 里漏了一条 `pass`，或 prefix-set 写成了默认拒绝

### 2.8.2 路由存在但转发不通

1. `ping` 失败但 `traceroute` 能到中途 → 数据平面
2. 源/目的两端确认 `show ip route` 都有该前缀
3. 路径中间逐跳 `show ip cef <prefix> detail`：看下一跳邻接是否 `resolved`、是否压标签
4. 大包不通 → 测 MTU（MPLS 场景把标签栈算进去）
5. 都不通且 FIB 正常 → 查 ACL / uRPF / 策略路由，以及回程路径是否被黑洞路由吸收

> 判断口诀：**控制面看"有没有"，数据面看"通不通"**。两者同时异常时，先修控制面。

---

## 2.9 三平台实战用例

> 以下命令输出为**示意**（字段顺序与真实设备一致），IP / AS 见本节拓扑。三平台指 **IOS**、**IOS XR**、**NX-OS**。

### 拓扑

```
        AS 65100                              AS 65200
   R1 ───────────── R2 ────────────────── R3
   Lo0 192.168.1.1/32   Lo0 192.168.2.2/32   10.1.23.3/24
        Gi0/1 10.1.12.1/30   Gi0/2 10.1.23.2/30
        └── IBGP（用 Lo0 建立）──┘   └── EBGP ──┘
                                        通告 172.16.3.0/24
```

| 设备 | 平台 | 角色 |
|------|------|------|
| R1 | IOS | AS 65100 内部，IBGP 客户端 |
| R2 | IOS XR | AS 65100 边界（ASBR） |
| R3 | NX-OS | AS 65200，前缀源 |

---

### 用例 1：新建 EBGP 会话 Established，但一条路由都收不到

**症状**：昨天新开 R2–R3 互联，邻居起来了，`PfxRcd` 一直是 0；R3 侧也看不到自己发出的前缀被接受。

**步骤 1 — 确认会话状态（三平台）**

```
! R2 (IOS XR)
show bgp ipv4 unicast summary
Neighbor        Spk    AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down  St/PfxRcd
10.1.23.3         0 65200      48      51        0    0    0 00:12:03          0
```

```
! R3 (NX-OS)
show bgp ipv4 unicast summary
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.1.23.2       4 65100      51      48        0    0    0 00:12:01          0
```

**判读**：两边 `St/PfxRcd` 都是 0，处理消息数在增长 → **TCP 与 OPEN 都正常**，问题在 UPDATE 层，属于控制面第 4 类故障（"优选但未通告"/"未收到"）。

**步骤 2 — 看双方的策略与收发计数**

```
! R2 (IOS XR) —— 谁被拒绝，计数就在谁身上
show bgp ipv4 unicast neighbors 10.1.23.3
  BGP state = Established, up for 00:12:03
  Last read 00:00:02, last write 00:00:11
  Hold time is 90, keepalive interval is 30 seconds
  ...
  Prefixes Current:              0          0 (Accepted prefixes)
  ...
  Inbound route policy:   (not configured)
  Outbound route policy:  (not configured)
```

```
! R3 (NX-OS) —— 我到底发了没有？
show bgp ipv4 unicast neighbors 10.1.23.2 advertised-routes
! 输出为空（No advertised routes）
```

**根因**：R3 是 IOS XR。**XR 的 EBGP 邻居如果不显式关联 `route-policy`，入向 NLRI 会被丢弃、出向不通告任何 NLRI**——会话照样能建立，所以 summary 上完全看不出异常。R3 缺的就是这一条策略。

**步骤 3 — 修复（XR 侧）**

```
route-policy PASS-ALL
  pass
end-policy
!
router bgp 65200
 neighbor 10.1.23.2
  remote-as 65100
  address-family ipv4 unicast
   route-policy PASS-ALL in
   route-policy PASS-ALL out
  !
 !
!
```

**对照写法（若对端是 IOS / NX-OS，同一件事的配置）**

```
! IOS：默认不限制，等价于"放行一切"
router bgp 65200
 address-family ipv4
  neighbor 10.1.23.2 activate
  neighbor 10.1.23.2 route-map PASS-ALL out
!
route-map PASS-ALL permit 10
```

```
! NX-OS：必须显式 activate，但不需要默认策略
router bgp 65200
 address-family ipv4 unicast
  neighbor 10.1.23.2 remote-as 65100
   address-family ipv4 unicast
```

**步骤 4 — 验证（三平台一致）**

```
show bgp ipv4 unicast summary                 ! PfxRcd 应变为 1（172.16.3.0/24）
show bgp ipv4 unicast 172.16.3.0/24           ! 本地 best 路径，下一跳 10.1.23.3
show bgp ipv4 unicast neighbors 10.1.23.2 advertised-routes   ! R3 侧应能看到该前缀
```

> **教训**：三平台"默认策略"不同是跨厂商互通的第一大坑。IOS 默认全放行，XR 默认全拒绝且**静默**，NX-OS 需要显式 `activate`。新建跨厂商会话时，先把这张表记住。

| 行为 | IOS | IOS XR | NX-OS |
|------|-----|--------|-------|
| 未配置策略时的入向 | 全接受 | **全丢弃** | 全接受 |
| 未配置策略时的出向 | 全通告 | **全不通告** | 全通告 |
| AFI 是否需要显式激活 | 默认激活 IPv4 | 需要 | 需要 |
| 异常是否体现在 summary | 否 | **否（无告警）** | 否 |

---

### 用例 2：IBGP 路由在表里但不可用（下一跳不可达）

**症状**：R2 从 EBGP 学到 `172.16.3.0/24` 且已最优，R1 的 BGP 表里也能看到这条路由，但 `*>` 变成了 `*`（无 `>`），实际转发不通。

**步骤 1 — 三平台看"这条路由为什么没被优选"**

```
! R1 (IOS)
show bgp ipv4 unicast 172.16.3.0/24
BGP routing table entry for 172.16.3.0/24, version 12
Paths: (1 available, no best path)
  Not advertised to any peer
  Refresh Epoch 1
  65200
    10.1.23.3 from 192.168.2.2 (192.168.2.2)
      Origin IGP, metric 0, localpref 100, valid, internal
      rx pathid: 0, tx pathid: 0
```

```
! R1 (IOS CLI 等价判断)
show ip bgp 172.16.3.0
  *  172.16.3.0/24  10.1.23.3   0   100 0 65200 i
                                        ↑ 前面没有 >，且无 best
show ip route 10.1.23.3
% Network not in table
```

```
! R2 (IOS XR) —— 我发给邻居的下一跳是谁
show bgp ipv4 unicast 172.16.3.0/24
  Local label: 24040
  10.1.23.3 from 10.1.23.3 (10.1.23.3)
    Received Label 24041
    Origin IGP, metric 0, localpref 100, valid, external, best
```

**判读**：R1 上 `show ip route 10.1.23.3` 无路由 → **下一跳不可达**，BGP 会做"下一跳可达性检查"，失败的路由不参与优选（`valid` 但无 best）。

**根因**：R2 把从 EBGP 学来的路由转给 IBGP 邻居时，**默认不改变 NEXT_HOP**（EBGP 学到的下一跳是外部地址，AS 内部不可达）。这是 IBGP 排错中最常见的一条。

**步骤 2 — 修复（三平台语法对比）**

```
! R2 (IOS XR)，在指向 R1 的 IBGP 邻居上开启
router bgp 65100
 neighbor 192.168.1.1
  remote-as 65100
  update-source Loopback0
  address-family ipv4 unicast
   next-hop-self
  !
 !
!
```

```
! IOS 等价配置
router bgp 65100
 neighbor 192.168.1.1 remote-as 65100
 neighbor 192.168.1.1 update-source Loopback0
 address-family ipv4
  neighbor 192.168.1.1 activate
  neighbor 192.168.1.1 next-hop-self
```

```
! NX-OS 等价配置
router bgp 65100
 neighbor 192.168.1.1
  remote-as 65100
  update-source Loopback0
  address-family ipv4 unicast
   next-hop-self
```

**步骤 3 — 验证**

```
show bgp ipv4 unicast 172.16.3.0/24    ! 下一跳应变为 192.168.2.2，且出现 > 标记
show ip route 172.16.3.0               ! 应出现 "B 172.16.3.0/24 [200/0] via 192.168.2.2"
```

> 若拓扑里 IBGP 邻居用的是**物理接口地址**而非 Lo0，下一跳可能本来就可达、不会复现此故障；这也是"配置比较法"能迅速定位的原因——拿一台正常的边界设备 diff 邻居配置即可。

---

### 用例 3：小包通、大包不通（数据平面 MTU 问题）

**症状**：路由全对、BGP 会话 Established、`ping 10.1.23.3` 通，但业务 TCP 传输文件时卡死；BGP 偶尔因为 UPDATE 报文较大而 reset。

**步骤 1 — 用分片禁止探测 MTU（三平台语法不同，这一组最常记错）**

```
! IOS
ping 10.1.23.3 size 1472 df-bit repeat 5
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/1 ms

ping 10.1.23.3 size 1473 df-bit repeat 5
.....
Success rate is 0 percent (0/5)
```

```
! IOS XR（size 含 IP 头 + 载荷；do-not-fragment 对应 df-bit）
ping 10.1.23.3 size 1472 do-not-fragment count 5
Type escape sequence to abort.
!!!!!
Success rate is 100 percent (5/5)
```

```
! NX-OS（参数名是 packet-size，不是 size）
ping 10.1.23.3 packet-size 1472 df-bit count 5
5 packets transmitted, 5 packets received, 0.00% packet loss
```

**步骤 2 — 接口 MTU 与丢弃计数（三平台）**

```
! IOS
show interfaces GigabitEthernet0/2 | include MTU|drops|output errors
  MTU 1500 bytes, BW 1000000 Kbit/sec, DLY 10 usec,
  0 output errors, 0 collisions, 0 interface resets
  Total output drops: 0
```

```
! IOS XR
show interfaces GigabitEthernet0/0/0/2 | include "MTU|drops"
  MTU 1500 bytes, BW 1000000 Kbit/sec
  0 output drops, 0 output errors
```

```
! NX-OS
show interface Ethernet1/2 | include MTU|drops
  MTU 1500 bytes, BW 1000000 Kbit, DLY 10 usec
  0 output drops
```

**判读**：1472 字节能通、1473 不能通 → 路径 MTU 恰好 1500（1472 + 20 IP + 8 ICMP）。两端接口 MTU 一致，但**中间链路或隧道封装多占了字节**，导致 1500 字节报文在中间被丢弃。

**步骤 3 — 修复（三平台接口 MTU 对比）**

```
! IOS
interface GigabitEthernet0/2
 mtu 1500
 ip mtu 1500
```

```
! IOS XR
interface GigabitEthernet0/0/0/2
 mtu 1500
 ipv4 mtu 1500
```

```
! NX-OS
interface Ethernet1/2
 mtu 1500
```

> 若是 MPLS 核心，注意 **1500 字节业务 + 标签栈** 会超过接口 MTU：核心接口通常要 1504/1508，或对业务侧做 MSS 调整（IOS：`ip tcp adjust-mss 1360`，其余平台命令名随版本不同，以实际命令树为准）。

**步骤 4 — 验证**：`ping ... size 1472 df-bit` 与 `size 1500 df-bit` 均通；业务侧 TCP 大包不再卡顿。

---

### 用例 4：会话周期性 Down/Up（flapping）

**症状**：邻居每天断 3～5 次，每次几秒后自动恢复；业务表现为路由短暂缺失。

**步骤 1 — 三平台日志（同一事件，三种格式）**

```
! IOS
%DUAL-5-NBRCHANGE: ...
%BGP-5-ADJCHANGE: neighbor 10.1.23.3 Down BGP Notification sent
%BGP-5-ADJCHANGE: neighbor 10.1.23.3 Up
```

```
! IOS XR
RP/0/RP0/CPU0:Oct 28 10:12:03.123 UTC: bgp[1051]: %ROUTING-BGP-5-ADJCHANGE : neighbor 10.1.23.3 Down - EOF received
RP/0/RP0/CPU0:Oct 28 10:12:06.881 UTC: bgp[1051]: %ROUTING-BGP-5-ADJCHANGE : neighbor 10.1.23.3 Up
```

```
! NX-OS
2026 Oct 28 10:12:03 bgp-65100 [1234]: %BGP-5-ADJCHANGE: bgp-65100 [1234] (default) neighbor 10.1.23.3 Down - BGP Notification sent
2026 Oct 28 10:12:06 bgp-65100 [1234]: %BGP-5-ADJCHANGE: bgp-65100 [1234] (default) neighbor 10.1.23.3 Up
```

**步骤 2 — 看"最后一次复位原因"，这是 flapping 的唯一硬证据**

```
! IOS
show bgp ipv4 unicast neighbors 10.1.23.3 | include "BGP state|Last reset|hold time|Last read"
  BGP state = Established, up for 00:00:12
  Last read 00:00:02, last write 00:00:11, hold time is 90, keepalive interval is 30 seconds
  Last reset 00:03:41 ago, due to Hold Timer Expired

! IOS XR
show bgp ipv4 unicast neighbors 10.1.23.3 | include "BGP state|Last reset"
  BGP state = Established, up for 00:00:12
  Last reset: 00:03:41, due to BGP Notification received

! NX-OS
show bgp ipv4 unicast neighbors 10.1.23.3 | include "BGP state|Last reset|hold time"
  BGP state = Established, up for 00:00:12
  Last reset by us 00:03:41, due to hold timer expired
```

**判读**：
- `Hold Timer Expired` → 对端 90 秒内没收到你的 keepalive ⇒ **路径丢包或对端 CPU/CoPP 丢包**，不是策略问题
- `BGP Notification received` → 对端主动发 Notification，去**对端**查日志与 max-prefix / capability 错误

**步骤 3 — 临时缓解：调整 keepalive/hold（三平台）**

```
! IOS
router bgp 65100
 address-family ipv4
  neighbor 10.1.23.3 timers 10 30
```

```
! IOS XR（timers 在 address-family 下）
router bgp 65100
 neighbor 10.1.23.3
  address-family ipv4 unicast
   timers 10 30
```

```
! NX-OS
router bgp 65100
 neighbor 10.1.23.3
  address-family ipv4 unicast
   timers 10 30
```

> ⚠️ 缩短 hold time 会让会话对丢包**更敏感**，只是把"90 秒发现"变成"30 秒发现"，属于掩盖症状。真根因要去查：接口 CRC/丢包计数、链路利用率、控制面 CPU、CoPP 是否限速过紧（BGP 属于 control-plane 流量）、以及是否被中间设备按会话老化。

**步骤 4 — 长期治理**：把 `Up/Down` 次数、`Last reset` 原因纳入监控，配合 EEM/SNMP trap 在第一次 flap 就留证据。

---

### 三平台诊断动作对照（背下这张表即可开工）

| 排查动作 | IOS | IOS XR | NX-OS |
|---------|-----|--------|-------|
| 会话总览 | `show bgp ipv4 unicast summary` | 同 IOS | 同 IOS |
| 复位原因 | `show bgp ... neighbors IP` → `Last reset ... due to` | 同 IOS | `Last reset by us / by peer` |
| 大包探测 | `ping IP size 1472 df-bit` | `ping IP size 1472 do-not-fragment` | `ping IP packet-size 1472 df-bit` |
| 接口 MTU | `show interfaces X \| include MTU` | `show interfaces X \| include "MTU"` | `show interface X \| include MTU` |
| 策略默认行为 | 全放行 | **全拒绝（静默）** | 全放行，AF 需 activate |
| 定时器修改位置 | `address-family` 下 | `neighbor → address-family` 下 | `neighbor → address-family` 下 |
| 抓 BGP 报文 | SPAN + 外部抓包 | `monitor capture` | `ethanalyzer local interface inband` |

### 小结：四个用例对应的方法论

| 用例 | 用到的排错方法 | 关键判据 |
|------|--------------|---------|
| 1. 收到 0 条路由 | 分而治之（TCP → OPEN → UPDATE 逐层切） | 消息数在涨但 PfxRcd 为 0 |
| 2. 下一跳不可达 | 沿路径追踪 + 配置比较 | `show ip route <下一跳>` 为空 |
| 3. 大包不通 | 自底向上（MTU 属于数据面） | 1472 通、1473 不通 |
| 4. 会话 flapping | 时间轴对齐日志 | `Last reset ... due to` |

> 这四类覆盖了 BGP 工单的绝大多数：**收不到、用不了、通不了、稳不住**。第十二章之后的所有专题，都是在这四个骨架上长细节。

---

## 关键命令速查表

| 命令 | 用途 |
|------|------|
| `show bgp ipv4 unicast summary` | 邻居总览，第一时间看状态与前缀数 |
| `show bgp ipv4 unicast neighbors IP` | 邻居细节：Accepted、策略、计时器、能力 |
| `show bgp ipv4 unicast <prefix>` | 该前缀的所有路径与落选原因 |
| `show bgp ipv4 unicast neighbors IP advertised-routes` | 验证出向通告 |
| `show bgp ipv4 unicast neighbors IP received-routes` | 验证入向接收（需 soft-reconfig） |
| `show ip route <prefix>` / XR: `show route <prefix>/len` | 路由是否进了 RIB |
| `show ip cef <prefix> detail` | 是否进 FIB、邻接与标签栈 |
| `show bgp ipv4 unicast update-group` | 更新组是否异常，判断通告是否被拆分 |
| `show logging` | ADJCHANGE / NOTIFICATION / MAXPFX |
| `show tech-support bgp` | 一次性收集，提交 TAC 用 |

---

## 三平台差异速查

| 特性 | IOS | IOS XR | NX-OS |
|------|-----|--------|-------|
| 路由表命令 | `show ip route` | `show route`（必带 /len） | `show ip route` |
| 接口表命令 | `show ip interface brief` | `show ipv4 interface brief` | `show ip interface brief` |
| 日志命令 | `show logging` | `show logging` | `show logging log` |
| BGP debug | `debug ip bgp updates` | `debug bgp update` | `debug bgp updates` |
| 内置抓包 | 无（需 SPAN） | `monitor capture` | `ethanalyzer` |
| 软重置 | `clear ip bgp IP soft in/out` | `clear bgp ipv4 unicast IP soft in/out` | `clear bgp ipv4 unicast IP soft in/out` |
| 条件化日志 | `logging discriminator` | `logging filter` | `logging level` |

---

## 本章核心要点

1. **方法论的本质是顺序**：定义问题 → 收敛范围 → 验证假设 → 修复回归，四步缺一不可
2. **第一个问题永远是"之前正常吗"**：从没通过查配置，曾经通过查变更
3. **一次只改一个变量**，否则永远学不到因果
4. **先分平面再看协议**：路由有而不通是数据面，无路由是不通是控制面
5. **控制面四检查点**：邻居未建立 → 入向被拒 → 未优选 → 未通告，按顺序排除
6. **`PfxRcd` 是最廉价的监控指标**，前缀数突变几乎总能定位到一次事件
7. **ping 应答走 CPU**，受 CoPP 限速，ping 丢包不能直接等价业务丢包
8. **debug 是最后手段**，生产上先关 console、开 buffer、限时、用完立刻关
9. **`clear ip bgp *` 会销毁证据**，除非明确需要重置，否则别动
10. **配置比较找语义差异**，不要执着于文本相同
11. **跨厂商第一坑是默认策略不同**：XR 未关联 `route-policy` 时静默丢弃入向、不通告出向；IOS/NX-OS 默认放行，NX-OS 另需显式 `activate`
12. **四类工单四种判据**：收不到看 `PfxRcd`、用不了看 `show ip route <下一跳>`、通不了测 1472 字节、稳不住看 `Last reset ... due to`

---

[← 第 1 章 · BGP 基础](chapter-01-bgp-fundamentals.md) | [← 返回首页](../) | 第 3 章 · 排查对等体问题（待精读）
