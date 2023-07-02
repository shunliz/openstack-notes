BGP（Border Gateway Protocol，边界网关协议）是一种被设计出来应用于 Internet 中的 Distance Vector 类型动态路由协议，能够在不同的 AS（Autonomous System，自治系统）间交换 Route Informations（路由信息）。因为 BGP Router 通常被部署在不同的 AS 之间的边界上，故命名为 “边界网关”。

BGP 协议起源于 1989 年 1 月举行的第 12 次 IETF 会议。在那时，由于 Internet 的快速发展，使得 Internet 中的各种网络的数量（AS 数量）快速增加，早期的手动静态路由配置方式显然已经无法满足在大量的 AS 之间管理路由信息。因此，会议上讨论的主题就是需要一种新的 EGP 动态路由协议。

![](/assets/network-basic-route-bgp1.png)

AS（自治系统）是 Internet 的基本定义之一，指的是一个逻辑上自包含的、自洽的 IP 网络系统，不同的 AS 之间可能运行着各自不同的 IGP 路由协议。

Internet 上的每个 AS 都具有一个 ASN（AS Number）作为唯一标识，全球 ASN 由 IANA（Internet Assigned Numbers Authority，互联网分配号码管理局）统一管理和分配，共有 1-65535 多个，代码体现为一个长度 16bits 的数字。其中：

* **1～64511**
  ：是全球唯一的 Internet 编号。
* **64512～65535**
  ：是自用的编号，作用类似于 IP 私网网段。

如下图所示，每个在 Internet 上提供网络服务的 AS（例如：运营商、大学、政企网络等）都需要拥有自己的 AS Number。在 BGP 协议出现之前，这些 AS 就像是一座座孤岛，与外界隔离。IETF 第 12 次会议的目的就是为了将这些大量的这些 AS 连接起来，其主要成果就是 BGP 协议。

![](/assets/network-basic-route-bgp2.png)

在会议结束后，由 Len Bosack、Kirk Lougheed 和 Yakov Rekhter 等人在餐巾纸上完成了 BGP 协议的设计草稿，为了解决 2 个 AS 之间的互联互通问题，其最初的设计也比较简单，归纳为以下 5 个想法：

1. 为了连接不同的 AS，应该在 2 个 AS 中分别部署 Border Gateway Router（边界网关路由器），由它们专门负责在不同的 AS 之间交换 Routes。
2. 为了避免在多个 AS 之间形成路由环路，应该在 Routes 中包含特定的 Path Attribute（路径属性），以此来确定 AS 之间的最优路径。
3. 为了在 WAN 中可靠的交换 Routes，应该使用 TCP 作为传输层协议。
4. 为了减少全球 Border Gateway Router 之间需要交换的海量 Routes，应该采用增量同步的交换方式。
5. 使用 TLV（Type-Length-Value）数据编码方式来定义 Message 的数据结构，使其拥有更好的功能可扩展性。

![](/assets/network-basic-route-bpg3.png)

最终在 1989 年 6 月发布了 RFC1105 BGPv1 标准。经过多年的发展后，现如今被广泛应用的是 BGPv4 版本，已经具有了以下完备的功能特性：

1. 支持 IPv4 和 IPv6；
2. 支持 CIDR（Classless Inter-Domain Routing）；
3. 支持 Multi-Path（多路径），提高网络的可用性和容错能力；
4. 支持 BGP Confederations（联盟）；
5. 支持 BGP Route-Reflectors（路由反射器）；
6. 支持 BGP Community（团体属性）；
7. 支持 BGP Route Dampening（路由惩罚）；
8. 支持 BGP MP（Multi-protocols Extensions，多协议扩展）；
9. 支持 Capability Advertisement（能力通告）；
10. 支持 BGPSEC 安全协议；
11. 等等。

虽然 BGP 最初定位于 EGP 场景，将 AS 作为距离度量单位，并通过强大的路由控制手段（例如：路由策略、路由过滤）来计算出 AS 之间的最佳路径。后来，随着 BGP 优秀的可扩展性也逐渐完善了在 IGP 场景中的应用，支持将 AS 内部的 Routers 作为距离度量单位，支持在一个大规模的 AS 内的所有 Router 之间发现、通告和计算 Routes。

区别于 OSPF、ISIS 等 Link State 类型路由协议，BGP 在大规模的 IGP 场景中能够基于强大的路由控制特性提供更好的网络稳定性（路由计算准确性高、路由收敛速度快）。举例来说，在大规模 IGP 组网中，任何路由节点发生故障时，OSPF 和 ISIS 都会引发整网的链路状态信息的泛洪和 LSDB 信息更新，然后在此基础上完成路由收敛。而 BGP 则只需要在特定的路有节点间通告路由，并通过增量同步的方式刷新路由信息，同时还具有路由域分区独立，故障域可控等优势。

## BGP Router 和 Routes

BGP 组网的核心就是 BGP Router，实现了 BGP 协议标准。能够对外 Advertise（发布）BGP Msg 的 BGP Router，称为 BGP Speaker（宣告者）。建立了 BGP Connection/Session 并互相交换 BGP Msg 的 Speaker 之间互称为 BGP Peers（对等体），根据应用场景的不同，又可以细分为 I-BGP Peer 和 E-BGP Peer，同时若干相关的 Peer 还可以构成一个 Peer Group（对等体组）。

而 BGP Routes 就被包含了专门的 BGP Update Msg 类型中。所以，BGP 协议实际上是一种承载在 TCP 协议之上的应用层协议。

![](/assets/network-basic-route-bgp5.png)

一个 Router 最基本的组成部分就是 RIBs（Routing Information Base）和 FIB（Forwarding Information Base）Table，前者负责 Control Plane 的路由选择，后者负责 Data Plane 的报文转发。

更具体的，BGP Router 的 RIB 由以下部分构成：

1. **Adj-RIBs-In**：用于存储从 BGP Peers 接收到的 Update Msg 中所携带的 Routes。或者根据 Update Msg 中的 WITHDAWN Route 在 Adj-RIBs-In 中删除相关条目。并在此后交由 Input Policy 处理。

2. **Input Policy**：当 Adj-RIBs-In 存储了从 BGP Peer 传递过来的 Routes 时，会根据本地的 Input Policy 并结合 Local RIB 中的内容来判断是否接受，只有满足路由策略的 Routes 才会被写入到 Router 的 RIB。例如：如果 BGP Router 收到 2 条 Routes，它们的目的网络一样，但是路径不一样，一个是 AS1=&gt;AS3=&gt;AS5，另一个是 AS1=&gt;AS2。那么，通常情况下，Router 会优先选用路径短的 AS1=&gt;AS2 这条 Route。

3. **BGP Selection Process（路径决策进程）**：再将 Routes 写入 Local RIB 之前，还需要经过 BGP Selection Process 进行处理。例如：将自己的 AS Number 注入到 Route 中，将 Next hop 改为自己，并将自己加入到 Path Attribute 中，形成一条新的可达信息。在这之后，这条信息会继续向其他 Peers 宣告，使得其他 Peers 知道可以通过 Next hop 到当前 Router，并最终到达目的网络。

4. **Local RIB**：用于存储 BGP Selection Process 的处理结果，同时某些本地路径也可以注入到 Local RIB 中。这些结果将用于生成 Local Route Table。

5. **Output Policy**：BGP Router 通过 Output Policy 来控制那些 Routes 是需要且允许对外进行宣告的。Local RIB 存储的结果在进行了一些 Output Policy 处理后，再把允许输出的 Routes 存储到 Adj-RIB-Out 中。

6. **Adj-RIB-Out**：最终 BGP Router 根据 Adj-RIB-Out 的结果向其它 Peers 发送 Update Msg。

另外，如果 BGP Router 收到的一条 Route 的 Path Attribute 中包含了自己的 AS Number，那么 Router 就会判定为这是一条自己发出的  Route，就会将这条 Route 丢弃掉。

![](/assets/network-basic-route-bgp6.png)

## BGP Message 类型和格式

BGP Message（消息），由 Header 和 Data 这两部分组成，最大长度为 4096Bytes。BGP Message 类型和格式的细节有很多，具体建议浏览相应的 RFC 文档，下面只作概括性的介绍。

![](/assets/network-basic-route-bgp7.png)

所有 BGP Msg Header 的格式都一样，共有 19Bytes。

* **Marker（16Bytes）**
  ：记录着同步信息和加密信息，用于检查 BGP Peer 的同步信息是否完整，以及用于 BGP 验证的计算。不使用验证时为全 1。
* **Length（2Bytes）**
  ：记录 BGP Msg 的总长度，长度范围是 19～4096。
* **Type（1Byte）**
  ：表示当前 BGP Msg Data 的类型。其取值从 1 到 5，分别表示下列消息类型：
* 1. Open Msg：用于对等体参数协商；
  2. Keepalive Msg：用于维护对等体邻居关系；
  3. Update Msg：用于通告可达路由和不可达路由；
  4. Notification Msg：用于错误信息通告，断开对等体邻居；
  5. Route-refresh：用于请求对等体重新发送路由信息。

### BGP Msg Data

#### Open Msg

Open Msg 是 TCP connection 建立后发送的第一个 BGP Msg 类型，用于建立 BGP Peers 之间的 Session 关系。

* **Version**
  ：指示 BGP 协议版本，通常为 4；
* **My AS**
  ：指示本地的 AS Number；
* **Hold Time（保持时间）**
  ：在建立 BGP Peer 关系时双方需要协商保持时间，如果在这段时间内未收到对端发来的 Keepalive Msg 和 Update Msg，则认为 BGP 连接中断了；
* **BGP Identifier**
  ：BGP 标识符，IP 地址形式，用来识别一个 BGP Router。

#### Keepalive Msg

Keepalive Msg 用于检测和维护 BGP Session 的健康状况，BGP Peers 之间会周期性地发出 Keepalive Msg，用来保持 Session 的有效性。

#### Update Msg

Update Msg 用于在 BGP Peers 之间交换 Routes，它既可以用于发布 Routes，也可以用于撤销不可达的 Routes。

Update Msg 是最关键的 BGP Msg 类型之一，BGP Routers 的 NLRI（Network Layer Reachability Information，网络层可达性信息）和 Path Attribute（路径属性）都被包含在里面。

* **Withdrawn Routes Length**
  ：指示 Withdrawn Routes 字段的长度。其值为零时，表示没有需要撤销的路由。
* **Withdrawn Routes**
  ：指示需要撤销的路由。
* **Total path attribute length**
  ：指示 Path Attributes 字段的长度。
* **Path Attributes**
  ：BGP Router 使用 Path Attribute 来确定前往目的地的最佳路径。
* **NLRI**
  ：BGP Router 使用 NLRI 中的 IP Prefixes（网络前缀）信息来完成路由分发。

两个 AS 的 BGP Router 之间通过 Update Msg 交换各自的网络信息，包括：IP Prefix、子网掩码和其他网络相关信息。

![](/assets/network-basic-route-bgp21.png)

#### Notification Msg

Notification Msg 用于 BGP Router 运维信息的通知，例如：当 BGP Router 检测到错误状态时，就会向 Peer 发出 Notification Msg，并中断 BGP Session。

* **Error Code、Error Subcode**
  ：指示错误码、错误子码，用于描述错误类型；
* **Data**
  ：具体的错误内容。 

#### Route-refresh Msg

Route-refresh Msg 用于要求 BGP Peer 重新发送指定地址族的 Routes，常用于实现路由刷新。

通常的，在 BGP Router 改变了自身的路由策略（Input/Output Policy）后就会请求 BGP Peers 重新发送 Routes。以此来实现 Peers 之间动态的交换路由刷新请求，并在后续的过程中使相关的 Adj-RIB-Out 重新通告路由。

### BGP Msg 状态机

![](/assets/network-basic-route-bgp22.png)**Idle（空闲状态）**：为初始状态，该状态下，BGP Router 拒绝 Peer 发送的连接请求。只有在收到本设备的 Start 事件后，BGP Router 才开始尝试和其它 BGP Peers 进行 TCP Connection，并转至 Connect 状态。

1. * Start 事件是用户配置或重配置一个 BGP 连接时触发的。
   * 另外，无论 BGP 处于任何状态中，当收到 Notification Msg 或 TCP 拆链通知等 Error 事件后，BGP 都会转至 Idle 状态。
2. **Connect（连接状态）**：BGP Router 启动 Connect Retry（连接重传定时器），等待 TCP Connection 完成。

3. * 如果 TCP 连接成功，那么 BGP Router 向 Peer 发送 Open Msg，并转至 OpenSent 状态；
   * 如果 TCP 连接失败，那么 BGP Router 转至 Active 状态。
   * 如果 Connect Retry 超时，BGP 仍没有收到 Peer 的响应，那么 BGP Router 会继续尝试和其它 BGP Peer 进行 TCP 连接，停留在 Connect 状态。
4. **Active（行动状态）**：该状态下 BGP Router 总是在试图与邻居建立 TCP Connection。

5. * 如果 TCP 连接成功，那么 BGP Router 向 Peer 发送 Open Msg，并关闭 Connect Retry，然后转至 OpenSent 状态。
   * 如果 TCP 连接失败，那么 BGP Router 停留在 Active 状态。
   * 如果 Connect Retry 超时，BGP 仍没有收到 Peer 的响应，那么就转到 Connect 状态。
6. **OpenSent（发送状态）**：BGP Router 等待 Peer 的 Open Msg，并对收到的 Open Msg 中携带的 AS Numer、Version、认证码等字段进行检查。如果收到的 Open Msg 正确，那么 BGP Router 发送 Keepalive Msg，并转至 OpenConfirm 状态。

7. **OpenConfirm（确认状态）**：BGP Router 等待 Keepalive 或 Notification Msg。

8. * 如果收到 Keepalive Msg，则转至 Established 状态；
   * 如果收到 Notification Msg，则转至 Idle 状态。
9. **Established（连接建立状态）**：BGP Router 建立了邻居关系后，就可以和 Peer 交换 Update、Keepalive、Route-refresh、Notification Msg 了。如果收到 Update 或 Keeplive Msg，则继续保持该状态；如果收到 Notification Msg，则迁移到 Idle 状态。

更具体的状态机流程如下图所示。

# ![](/assets/network-basic-route-bgp23.png)BGP Path Attributes 与路由选择

这里单独展开 BGP Update Msg 中的 Path Attributes 与 BGP Router 进行路由选择之间的关系。总体而言，BGP Path Attributes 可以分为以下 4 大类型。

### 公认必遵属性（Well-known mandatory）

是所有的 BGP Router 都能够识别该属性，并且必须出现在所有 Update Msg 中。包括：

1. **ORIGIN（源头）**：指出了 BGP Routes 的来源，用于判断 Routes 的可信度，Router 会根据 ORIGIN 属性作为路由决策的参考。可以是以下 3 种值，在路由选择的时候，IGP 优于 EGP，EGP 优于 INCOMPLETE。

2. 1. IGP：表示网络层可达信息来源于 AS 内部。
   2. EGP：表示网络层可达信息通过 AS 外部学习。
   3. INCOMPLETE：表示网络层可达信息来源无法确定。
3. **AS\_PATH（AS 路径）**：它通过一种 Record-Route（记录路由）的方式，记录了一个 IP Prefix（路由前缀）在传递过程中所经过了的 AS。采用 AS\_SEQUENCE 方式表示，即该路由经过的 AS 的有序集合。当 BGP 发布者发布路由给 IBGP 对等体时，BGP 不修改路由的 AS\_PATH 属性。当 BGP 发布者发布路由给 EBGP 对等体时，本地系统应该把自己的 AS 号作为序列的最后一个元素加在序列的最后面。所以，AS\_PATH 可以用来作为路由选路的一种度量。经过更少 AS 路径的路由更优先。同时，AS\_PATH 也作为一种手段来避免环路。如果 BGP 路由信息发布者从 E-BGP 对等体收到一条路由，它的 AS\_PATH 包含发布者自己的 AS 号，就说明这条路由曾经从本 AS 发出过，将其丢弃，同时不再进行转发。基于上述机制，AS\_PATH 属性可以避免 AS 之间的路由环路的出现，AS 内部的路由环路的避免则采用其他手段来实现。

4. **NEXT\_HOP（下一跳）**：表示目的网络所使用的下一跳路由器的 IP 地址。如果是发布给 EBGP 对等体，NEXT\_HOP 填写 BGP 发布者的 IP 地址。如果是发布给 IBGP 对等体，且路由来自 AS 外部，则 NEXT\_HOP 保留原始的 AS 外部对等体的 IP 地址。

### 公认可选属性（Well-known discretionary）

是所有的 BGP Router 都能够识别该属性，但可以不出现在 Update Msg 中。包括：

1. **LOCAL\_PREF（Local Preference，本地优先级）**：Update Msg 可以携带这个属性并将其发给 I-BGP 邻居，用于 AS 内部的 BGP Router 作为参考，具有较高的 LOCAL\_PREF 值的 Routes 将在路由选择过程中被优先考虑。仅在 I-BGP 对等体之间交换，不通告给其他 AS。当 BGP 的路由器通过不同的 IBGP 对等体得到目的地址相同但下一跳不同的多条路由时，将优先选择 LOCAL\_PREF 属性值较高的路由。如下图所示，Router B 和 Router C 发给 Router D 的关于 8.0.0.0 的路由携带不同的 LOCAL\_PREF 值，从而引导从 AS 20 到 AS 10 的流量将选择 Router C 作为出口。

![](/assets/network-basic-route-bgp25.png)

**ATOMIC\_AGGREGATE（原子聚合）**：当 BGP Router 进行路由聚合时，由于会产生一条新的聚合路由，因此精细路由所携带的 AS-path 属性将会在聚合时被丢失。用来通告路由接收者，该路由是经过聚合的。有时 BGP 发布者会收到两条重叠的路由，其中一条路由包含的地址是另一条路由的子集。一般情况下 BGP 发布者会优选更精细的路由（前者），但是在对外发布时，如果它选择发布更粗略的那条路由（后者），这时需要附加上 ATOMIC\_AGGREGATE 属性，以知会对等体。它实际上是一种警告，因为发布更粗略的路由意味着更精细的路由信息在发布过程中丢失了。在进行路由聚合时，对于聚合的路由信息会添加 ATOMIC\_AGGREGATE 属性。

### 可选传递属性（Optional transitive）

不要求所有 BGP Router 都能识别，但即使不能识别也会传递该属性。包括：

1. **AGGREGATOR（聚合站点）**：可以包含在产生聚合路由的 Update Msg 中，通过携带发送 Update Msg 的 Router 的 BGP-ID，以此来告知进行了路由聚合通告的 Router 的标识。是 ATOMIC\_AGGREGATE 属性的补充。ATOMIC\_AGGREGATE 是一种路由信息丢失的警告，AGGREGATOR 属性补充了路由信息在哪里丢失，即它包含了发起路由聚合的 AS 号码和形成聚合路由的 BGP 发布者的 IP 地址。在进行路由聚合时，当对于聚合的路由信息同添加 ATOMIC\_AGGREGATE 属性的同时，会添加 AGGREGATOR 属性。

2. **Community（共同体）**：在 RFC1997 和 RFC1998 中定义，用于对 Routes 进行分组管理。通常在制定路由策略时会对一系列的 IP Prefix 进行控制，例如：对从某个 AS 来的 Routes 进行特殊处理等。基于这样的原因，可以通过在 Update Msg 中携带 Community 属性来进行相关的路由策略管理。例如：ISP 可以为某个特定的用户分配一个 Community 属性值，此后该 ISP 就可以基于 Community 值来设置专门的  LOCAL\_PREF 或者 MED 等属性来完成路由策略的控制。团体属性用来简化路由策略的应用和降低维护管理的难度，没有物理上的边界，与其所在的 AS 无关。公认的团体属性有：

3. 1. INTERNET：缺省情况下，所有的路由都属于 Internet 团体。具有此属性的路由可以被通告给所有的 BGP 对等体。
   2. NO\_EXPORT：具有此属性的路由在收到后，不能被发布到本地 AS 之外。如果使用了联盟，则不能被发布到联盟之外，但可以发布给联盟中的其他子 AS。
   3. O\_ADVERTISE：具有此属性的路由被接收后，不能被通告给任何其他的 BGP 对等体。
   4. NO\_EXPORT\_SUBCONFED：具有此属性的路由被接收后，不能被发布到本地 AS 之外，也不能发布到联盟中的其他子 AS。

### 可选非传递属性（Optional non-transitive）

不要求所有 BGP Router 都能识别，不识别该属性就会丢弃该 Msg。包括：

1. **MED（Multi-exit-discriminator，多出口鉴别器）**：用来区分同一个邻接 AS 的多个接口，用于在具有多个出口的 AS 之间选择最优路径，MED 值越小，路径的开销就越小，该路径将被优先选择。MED 只在 EBGP 发布的路由中产生，接收者可以向它的 IBGP 邻居转发，但不允许向它的 EBGP 对等体转发。假设一个 AS 和邻接 AS 有多个接口相连，通过发布不同的 MED 给对端，就可以控制进入网络的流量从 MED 值最小的那个接口进来。通常情况下，BGP 只比较来自同一个 AS 的路由的 MED 属性值。如下图所示，Router B 和 Router C 发给 Router A 的关于 9.0.0.0 的路由携带不同的 MED 属性，从而引导从 AS 10 到 AS 20 的目的地址为 9.0.0.0 网段的流量将选择 Router B 作为入口。

![](/assets/network-basic-route-bgp26.png)

1. **ORIGINATOR\_ID**：用于标识路由反射器，为了防止引入路由反射器之后出现环路，增加 ORIGINATOR\_ID 这个属性来标识，反射器在发布路由时加入 ORIGINATOR\_ID，当反射器收到的路由信息中的 ORIGINATOR\_ID 就是自己的 ROUTER\_ID 时，就可以发现路由环路的出现，将该路由丢弃，不再转发。

2. **CLUSTER\_ID**：用于标识路由反射器组，用来防止环路，在路由经过路由反射器时路由反射器会将自己的 CLUSTER\_ID 添加到路由携带的 CLUSTER\_LIST 中，当路由反射器发现接收的路由的 CLUSTER\_LIST 中包含有自己的 CLUSTER\_ID，则将该路由丢弃，不再转发。

### 路由选择原则

![](/assets/network-basic-route-bgp31.png)

如上图所示，在一个 Internet 中具有 6 个 AS。如果此时 AS1 需要向 AS3 路由一个 IP 数据包，那么它有两种不同的路径：

1. AS2 → AS3
2. AS6 → AS5 → AS4 → AS3

在这个简化的示例中，路由选择显然倾向于路径 1，它是最快、最高效的路由。然而在实际现网中，则基于 Path Attribute 的影响，BGP Router 进行具体的路由选择时，通常遵守下述规则：

1. **权重**
   ：首选 Weight 值最高的。
2. **本地优先级**
   ：如果 Weight 相同，选择 local-preference 最高的。
3. **本地始发**
   ：如果 local-preference 相同，选择本地发出的 Route（Next-hop=0.0.0.0），优于从 Peer 学习到的。
4. **AS 路径长度**
   ：如果没有当前路由器通告的路由，选择 AS-path 最短的。
5. **源地属性（Origin）**
   ：如果 AS-path 相同，选择 Origin 最优的（IGP 
   &gt;
    EGP 
   &gt;
    不完全）。
6. **MED**
   ：如果 Origin 相同，选择 MED 最低的。
7. **E-BGP over IBGP**
   ：如果 MED 相同，则 E-BGP 优于 I-BGP。
8. **E-BGP 到达顺序**
   ：若都是 E-BGP，选择优先到达的。
9. **I-BGP 下一跳开销**
   ：若都是 I-BGP，选择下一跳最近的。
10. **集群列表（Cluster list）**
    ：优先选择最短的 cluster-list，仅适用于 RR（路由反射器）客户端。
11. **Router ID**
    ：首选 BGP 邻居的 Router-id 最小的。
12. **对等体 IP 地址最小长度**
    ：如果 Router-id 相同，选择邻居 IP 地址最小的。

## BGP RR（Route-Reflectors，路由反射器）

在 BGP RR（Route-Reflectors，路由反射器）标准被提出之前，采用的是 BGP Full Mesh 组网拓扑，即：每一个 BGP Speaker 都需要和其他 BGP Speaker 建立 BGP Session。这样全网中 BGP Session 的总数就是 N^2，如果 BGP Cluster 的规模超过了 100 台，就会对设备造成非常大的配置和处理压力。因此 Tony Bates 和 Ravi Chandra 在 1996 年 6 月提出了《RFC1966: Route Reflector》标准。

BGP RR 通过指定少量的（一个或多个）高性能 BGP Speaker 作为 RR，由它们与网络中其他 BGP Router 建立 BGP Session，并负责将 Routes 信息反射给所有建立连接的 Peers。每个 BGP Router 只要成为了 RR 的 Peer 之后，即可获得全网的 Routes 信息，以此来有效减轻 Cluster 的配置压力。目前 BGP RR 已经被广泛应用在 I-BGP 和 E-BGP 场景中，例如：Kubernetes Calico CNI 等。

![](/assets/network-basic-route-bgp32.png)为了保证 RR 方案的简洁性和支持平滑升级，RR 标准引入了一些新的概念。比如，当一个 BGP Speaker 被配置为 RR 后，它会将 BGP 邻居分为 2 类：

1. **Client Peer**
2. **Non-Client Peer**

并且在宣告路由时遵循以下规则：

1. 当收到一个来自 Non-Client Peer 的 I-BGP Update Msg 后，该 Route 将会反射到所有的 Client Peer。
2. 当收到一个来自 Client Peer 的 I-BGP Update Msg 后，该 Route 将会反射到所有的 Client Peer 和 Non-Client Peer。
3. 当收到一个来自 E-BGP Update Msg 后，该 Route 将会反射到所有的 Client 和 Non-Client Peer。

## BGP MP（Multi-protocols Extensions，多协议扩展）

最初的 BGPv1 只能管理 IPv4 协议和单播路由，后来为了让 BGP 支持更多的协议类型（e.g. IPv6）和路由类型（e.g. 多播），在《RFC 4760: Multiprotocol Extensions for BGP-4》中定义了 BGP MP 标准。支持 MP 的 BGP Router 又称为 MP-BGP Router（多协议 BGP 路由器）。

BGP MP 的实现思路是，在 Update Msg 的 TLV 编码格式的基础上，对 NLRI 字段进行扩展，添加了协议类型字段和子网前缀字段，用来描述不同协议类型和路由类型信息。同时，为了区分 NLRI 中的多种协议类型和路由类型，还分别引入了 AFI（Address Family Identifier，地址族标识）和 SAFI（Subsequent Address Family Identifier，子地址族标识）的概念。  


通过这样的方式，BGP MP 的可扩展性得到了极大的增强，使得 BGP 已经不仅仅是一个单纯的动态路由协议，而是可以被设计用于支持一些新型的应用场景，例如：IPv6 单播地址族、MPLS VPN 地址族、EVPN VxLAN 地址族等等。

更具体的，MP-BGP 支持了以下多种协议类型：

* **IPv4 地址族**
  ：用于路由 IPv4 地址，是 BGP 的最原始地址族。
* **IPv6 地址族**
  ：用于路由 IPv6 地址。
* **VPNv4 地址族**
  ：用于 MPLS VPN 的 IPv4 路由。
* **VPNv6 地址族**
  ：用于 MPLS VPN 的 IPv6 路由。
* **L2VPN EVPN 地址族**
  ：用于 EVPN 的路由。
* **IPv4 Flow Label 地址族**
  ：用于标识 IPv4 Flow 的标签路由。
* **IPv6 Flow Label 地址族**
  ：用于标识 IPv6 Flow 的标签路由。

以及支持了下列多种路由类型：

* **Unicast（单播路由类型）**
  ：表示路由信息的目的地址只有一个。
* **Multicast（多播路由类型）**
  ：表示路由信息的目的地址有多个。
* **Flow（流路由类型）**
  ：它是一种带有源地址和目的地址的路由类型，用于 MPLS 标签转发流量工程。

## BGP Link-state

BGP-LS（BGP Link-state）是一种网络拓扑信息提取技术，用于汇总跨域（Area）或跨 AS 间的若干个 IGP 网络拓扑信息并上报给集中控制器。

BGP-LS 最初在 RFC 7752 中被定义，是一个可选的、非必需传递的 BGP 扩展。扩展了 BGP 使其可以携带与 SR 和性能参数相关的附加信息。BGP-LS 标准包含两部分：

1. BGP NLRI
2. BGP Path Attribute

在 BGP-LS 诞生之前，路由器可以通过使用 IGP 路由协议（e.g. OSPF、IS-IS）来收集网络的拓扑信息。这种拓扑信息提取方式，存在以下几点不足：

1. IGP 协议将各个 Area 的拓扑信息单独上送给上层控制器，在跨 IGP Area 的场景中，控制器无法得到完整的拓扑信息，所以也无法计算出跨 Area 的端到端最优路径；
2. 要求控制器支持 IGP 协议及其算法；
3. 控制器需要支持不同的 IGP 协议，导致控制器对拓扑信息的分析处理过程比较复杂。

BGP-LS 就是为了解决上述问题而诞生的。通过 BGP-LS，会把 IGP 协议发现的拓扑信息都汇总到 BGP 协议并上报给控制器。利用 BGP 协议强大的选路和算路能力，具有了以下几点优势：

1. BGP 协议可以汇总跨 IGP AS 域的拓扑信息，直接将完整的拓扑信息上送给控制器，有利于路径选择和计算；
2. 不要求控制器实现 IGP 协议及其算法；
3. 控制器仅仅需要支持 IGP 协议即可。

### BGP-LS Route Msg

BGP-LS 实现了 3 中 Routes，分别用来携带 Nodes、Links 和 Prefix（路由前缀）信息。3 种 Routes 相互配合，共同完成拓扑信息的提取。

#### Node Route

Node Route 用于记录网络拓扑中的节点信息。

Route 格式示例：

```
[NODE][ISIS-LEVEL-1][IDENTIFIER0][LOCAL[as100][bgp-ls-identifier11.1.1.2][ospf-area-id0.0.0.0][igp-router-id0000.0000.0001.00]]


```

Route 字段含义：

1. **NODE**
   ：标识此 BGP-LS Route 是 Node Route 类型。
2. **ISIS-LEVEL-1**
   ：标识收集拓扑信息的 IGP 协议类型为 ISIS。
3. **IDENTIFIER0**
   ：该 IGP 协议类型中的 Node Routes 的唯一标识。
4. **LOCAL**
   ：标识该 Node Route 为 Local Node 的信息。
5. **AS**
   ：标识 BGP-LS 的 AS Number。
6. **bgp-ls-identifier**
   ：标识 BGP-LS 的区域。
7. **ospf-area-id**
   ：标识 OSPF 的区域。
8. **igp-router-id**
   ：IGP 协议的 Router ID，由收集拓扑信息的 IGP 协议产生。

#### Link Route

Link Route 用于记录两台设备之间的链路信息。

Route 格式示例：

```
[LINK][ISIS-LEVEL-1][IDENTIFIER0][LOCAL[as255.255][bgp-ls-identifier192.168.102.4][ospf-area-id0.0.0.0][igp-router-id0000.0000.0002.01]][REMOTE[as255.255][bgp-ls-identifier192.168.102.4][ospf-area-id0.0.0.0][igp-router-id0000.0000.0002.00]][LINK[
if
-address0.0.0.0][peer-address0.0.0.0][
if
-address::][peer-address::][mt-id0]]


```

Route 字段含义：

1. **LINK**
   ：标识此 BGP-LS Route 是 Link Route 类型。
2. **ISIS-LEVEL-1**
   ：标识收集拓扑信息的 IGP 协议类型为 ISIS。
3. **IDENTIFIER0**
   ：该 IGP 协议类型中的 Node Routes 的唯一标识。
4. **LOCAL**
   ：标识该 Node Route 为 Local Node 的信息。
5. **AS**
   ：标识 BGP-LS 的 AS Number。
6. **bgp-ls-identifier**
   ：标识 BGP-LS 的区域。
7. **ospf-area-id**
   ：标识 OSPF 的区域。
8. **igp-router-id**
   ：IGP 协议的 Router ID，由收集拓扑信息的 IGP 协议产生。
9. **REMOTE**
   ：标识该 Route 的对端节点的信息。
10. **if-address**
    ：Local 接口地址。
11. **peer-address**
    ：Remote 接口地址。
12. **mt-id**
    ：在 IGP 协议中用于标识接口所绑定的拓扑。

#### Prefix Route

Prefix Route 用于记录 IP 可达的网段信息。

Route 格式示例：

```
[IPV4-PREFIX][ISIS-LEVEL-1][IDENTIFIER0][LOCAL[as100][bgp-ls-identifier192.168.102.3][ospf-area-id0.0.0.0][igp-router-id0000.0000.0001.00]][PREFIX[mt-id0][ospf-route-type0][prefix192.168.102.0/24]]


```

Route 字段含义：

1. **IPV4-PREFIX**
   ：标识此 BGP-LS Route 是 Prefix Route 类型，并且是 IPv4 Prefix Route，此外还支持 IPV6-PREFIX。注意，路由器不能本地产生 IPv6 地址前缀路由，但可以处理来自其他厂商的 IPv6 地址前缀路由。
2. **ISIS-LEVEL-1**
   ：标识收集拓扑信息的 IGP 协议类型为 ISIS。
3. **IDENTIFIER0**
   ：该 IGP 协议类型中的 Node Routes 的唯一标识。
4. **LOCAL**
   ：标识该 Node Route 为 Local Node 的信息。
5. **AS**
   ：标识 BGP-LS 的 AS Number。
6. **bgp-ls-identifier**
   ：标识 BGP-LS 的区域。
7. **ospf-area-id**
   ：标识 OSPF 的区域。
8. **igp-router-id**
   ：IGP 协议的 Router ID，由收集拓扑信息的 IGP 协议产生。
9. **PREFIX**
   ：标识一条 IGP Route。
10. **mt-id**
    ：在 IGP 协议中用于标识接口所绑定的拓扑。
11. **ospf-route-type**
    ：标识 OSPF 的路由类型。
12. 1. Intra-Area；
    2. Inter-Area；
    3. External 1；
    4. External 2；
    5. NSSA 1；
    6. NSSA 2。
13. **prefix**
    ：标识 IGP Route 的前缀地址。

### BGP-LS 应用示例

#### IGP Area 内的网络拓扑信息提取

如下图所示。A、B、C、D 之间通过 IS-IS 协议实现 IP 可达，并且同属于 Area10，都是 Level-2 设备。

在这种情况下，只需要 A、B、C 和 D 中的任何一台网络设备部署 BGP-LS feature（特性）并与控制器建立了 BGP-LS 邻居关系便可提取出整个网络的拓扑信息。

在 HA 场景中，会选择两台或以上的设备来实施。由于网络拓扑信息是相同的，所以它们之间可以互为备份。

![](/assets/network-basic-route-bgp34.png)

#### IGP Area 间的网络拓扑信息提取

如下图所示。A、B、C、D 之间通过 IS-IS 协议实现 IP 可达。其中，A、B 和 C 属于 Area10，D 属于 Area20。A、B 是 Level-1 设备，C 是 Level-1-2 设备，D 是 Level-2 设备。

虽然网络拓扑跨越了 Area10 和 Area20 这 2 个域，但 BGP-LS 支持提取整个网络的拓扑信息，所以依旧只需要一台网络设备部署了 BGP-LS feature 并与控制器建立 BGP-LS 邻居关系便可。

同样，HA 场景中，可以使用 2 台及以上的设备进行实施，它们互为备份。

![](/assets/network-basic-route-bgp35.png)

#### EGP AS 间拓扑信息提取典型组网 1

如下图所示。A 和 B 同属 AS 100，两者之间建立 IGP IS-IS 邻居关系，并且 A 作为 AS 100 内部的一台非 BGP 设备。B 和 C 之间建立 EBGP 连接。

在这种情况下，由于（未使能 BGP-LS 的）BGP 协议本身不能提取网络拓扑信息，所以在 AS 100 内的设备和在 AS 200 内的设备上收集的拓扑信息不同（都只能收集各自 AS 域的拓扑信息），所以此时要求至少 AS 100 和 AS 200 两个 AS 中都至少有一台设备使能 BGP-LS 特性并都与控制器建立 BGP-LS 邻居关系。

每个 AS 中有两台或以上设备与控制器相连则可以保证 HA。

![](/assets/network-basic-route-bgp36.png)  
EGP AS 间拓扑信息提取典型组网 2

若网络中存在两台控制器，分别与两个 AS 中的设备相连，如下图所示，此时若想两台控制器上都能收集到整个网络的拓扑信息，则需要两台控制器之间建立 BGP-LS 邻居关系或与控制器相连的 B 和 C 之间建立 BGP-LS 邻居关系。

此时，为了减少与控制器连接的数量，可以选择一台（或几台）设备作为 BGP-LS 反射器，需要与控制器建立 BGP-LS 邻居的设备都与反射器建立邻居关系。

![](/assets/network-basic-route-bgp37.png)

#### BGP-LS 在 SR TE 中的应用

在 SR TE 应用场景中，需要在数据库中储存 Node、Link、Prefix、SR Policy、Ingress/Egress Node（端节点）等一系列数据，用于 SDN Controller 进行路径计算和验证。

在 RFC4655 中，提出了 SR-PCE（SDN 控制器）组件，它具有整个网络的拓扑和链路状态信息数据库的全局视图。SR-PCE 通过 BGP-LS 获取所需要的信息（Node、Link、Prefix、SR Policy 等），以构建其 SRTE 数据库，以此计算出 SRTE 所需的域间路径。

每个 Area 的 Edge Router 都需要与 SR-PCE 建立 BGP-LS 对等互连，提供网络拓扑信息和链路状态信息，包括：IGP 度量、TE 度量、管理组、SRLG 等。

每个 Area 的本地信息通常来自 IGP（也可以是 BGP）协议，IGP 将本地信息汇总到 BGP-LS，然后通过 BGP NRLI（网络层可达性信息）传递给 SR-PCE。

