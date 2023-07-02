## 广域网络数据通信技术发展历程

自 1978 年温顿·瑟夫、鲍伯·卡恩、丹尼·科恩（Danny Cohen）和约翰·普斯特尔（Jon Postel）合力将 TCP 协议从分层思想的角度划分为 IP 和 TCP 这 2 个协议并建立了著名的沙漏模型以来。

IP 路由转发技术就被广泛的应用到了广域网络数据传输场景中。IP 技术是一种无连接的、尽力而为的通信方式，采用了 “最长匹配优先” 算法进行路由转发，每个 Router 都需要先解析出 IP Packet 的 dstIP 地址，并根据此信息独立对 Packet 进行路由决策。

上世纪 90 年代中期，IP 技术凭借实现简单和成本低廉的优势快速发展，但由于硬件转发芯片技术存在限制，Router 必须要提供软件实现的最长匹配优先算法，难免转发性能低下。因此 IP 技术的转发性能成为当时限制网络发展的瓶颈。高性能的快速路由器技术成为当时研究的热点之一。

ATM（Asynchronous Transfer Mode）技术就是在这样的时代背景下诞生的。ATM 源于 1983 年美国贝尔研究所提出的快速分组交换以及 1984 年法国电信 CENT 提出的异步时分交换思想，它采用了固定长度为 188Bytes 的长标签（信元）转发方式，并且只需要维护比路由表规模小得多的标签表，因此能够提供比 IP 路由方式高得多的转发性能。同时，ATM 还采用了面向连接和服务质量保障的方式进行通信。

由于 ATM 技术能够满足 ISP 对广域网络的可靠性和可管理性要求，因而在 ATM 技术兴起的早期就被广泛应用。然而，由于 ATM 协议相对复杂且部署成本非常高，这也使 ATM 技术很难普及。

随着 Internet 的爆炸式增长，经济性和效率性的边界效益随着网络的规模进一步增大而降低。如何更有效的建设更大规模的广域网络成为摆在面前的现实问题。此外，随着计算机网络向宽带化、智能化方向发展，网络业务也呈现出突发特性。网络通信更加强调效率和通用性。

ATM 以交换机为核心，通过信令技术建立长标签转发路径，强调网络智能。而 IP 以路由器为核心，通过路由技术支持逐跳转发，强调终端智能。ATM 与 IP 有着各自的特色，也都有着适合自己的应用场景。

* IP 技术简单，但性能受到限制；
* ATM 技术性能高，但协议相对复杂，且 ATM 网络部署成本高昂。

很自然的，在当时的背景下，如何将 IP 和 ATM 各自的优势结合起来是一个热门的研究主题，即：在保持 IP 技术简洁性的前提下，提供 ATM 技术的高性能。

直到，在 1996 年 MPLS（Multi-Protocol Label Switching，多协议标签交换技术）在这种背景下诞生。至此，传统网络中就拥有了 3 种经典转发实现，它们分别是：

* L2 交换转发
* L2.5 标签转发
* L3 路由转发

MPLS 协议则作用于 L2.5 层，其将 L3 路由技术和 L2 交换技术相结合，充分发挥了 IP 路由的灵活性和 MAC 交换的高效性能。

## MPLS 协议格式

### MPLS Label Stack

MPLS Label Stack 是 MPLS 协议格式的核心，又称之为 MPLS Header，其位于 L2 Header 和 L3 Header 之间，使得 MPLS 能够适用于 “Multi-Protocol（多种上层 Internet 协议类型）"，包括：IPv6、IPX（Internet Packet Exchange）和 CLNP（Connectionless Network Protocol）等等。

![](/assets/network-basic-route-mplss1.png)

  


如上图所示，MPLS Label Stack 由若干个 MPLS Label Entries 组成。这些 MPLS Label Entries 从位置上还可以分为：

* **Outer MPLS Label（栈顶标签）**
  ：靠近 L2 Frame Header 的 Label Entry。
* **Inner MPLS Label（栈底标签）**
  ：靠近 L3 Packet Header 的 Label Entry。

![](/assets/network-basic-route-mplss2.png)

  


多条 Label Entries 遵循着 FIFO（先进先出）的 Stack 特性，也就说 MPLS 设备会从 Outer Label 开始进行处理，此时的 Stack Index 为 0。直到 Stack Index 为 1 时，指向了 Inner Label，此时 MPLS 设备再执行一次 POP 操作，就完全剥除 MPLS Label Stack，还原了一个标准的 IP Packet。

![](/assets/network-basic-route-mplss3.png)

  


如下面的 2 个例子，它们分别包含了 2 条和 3 条 Entries。

* 2 Label Entries
  ![](/assets/network-basic-route-mplss4.png)
* 3 Label Entries
  ![](/assets/network-basic-route-mplss5.png)

MPLS Label Stack 是实现 MPLS-TE FRR（Traffic Engineering Fast ReRoute）或 SR-MPLS 的关键，它们可以被 SDN Controller 统一控制标签转发的路径。

![](/assets/network-basic-route-mplss6.png)

  


### MPLS Label Entries

![](/assets/network-basic-route-mplss7.png)上图是一条 MPLS Label Entry 的报文各式，包括了：

* **Label（20bits）**
  ：用于标识一个 FEC 或作为仅在设备本地有特殊意义的标识，支持百万级别的容量。Label Space 的标准定义如下：
* * 0-15：特殊标签值。
  * 16-1023：静态信令协议的标签空间，例如：静态 LSP、静态 CR-LSP 等。
  * 1024-1048575：动态信令协议的标签空间，例如：LDP、RSVP-TE、MP-BGP 等。动态信令协议的标签空间不是共享的，而是独立且连续的，互不影响。
* **EXP（3bits）**
  ：扩展字段，现常被用于 CoS（Class of Service）特性。
* **S（Bottom of Stack，1bit）**
  ：在 MPLS Label Stack 场景中表示该 Entry 的位置。
* **TTL（生存时间，8bits）**
  ：当 IP Packet 被封装到 MPLS Header 之后，IP Header TTL 会被 Copy 到 MPLS Header TTL。然后，每一次执行 MPLS LSH，MPLS Header TTL - 1。当解封装 IP Packet 时，再将 MPLS Header TTL Copy 到 IP Header TTL。

针对将 EXP 字段应用于 MPLS TC（Traffic Control， 流量控制）场景，在 RFC 5462 中，将 EXP 字段被定义为了 TC 字段，用于支持 QoS and ECN 的功能。如下图所示。

![](/assets/network-basic-route-mplss9.png)

MPLS 广域网络转发原理

### MPLS 的基本组网元素

![](/assets/network-basic-route-mplss21.png)

  


* **LSR（Label Switching Router，标签交换路由器）**
  ：MPLS 网络的 Core 设备。
* **LER（Label Edge Router，标签边缘路由器）**
  ：MPLS 网络的 Network Edge 设备。可在细分为 2 类：
* 1. **MPLS Ingress Node**
     ：进入 MPLS 网络的 LER，Ingress Node 会计算出 IP Packets 归属的 FEC，并把 FEC 对应的 Label 封装到 IP Packet。
  2. **MPLS Egress Node**
     ：出 MPLS 网络的 LER，将 MPLS Label 剥离，还原为标准的 IP Packet 再重新回到 IP Routing 网络中。

![](/assets/network-basic-route-mplss22.png)

  


### MPLS Router 的基本组成部分

![](/assets/network-basic-route-mplss23.png)

  


MPLS Control Plane 负责管理 Routes 和 Labels 信息等。

* **IRP（IP Routing Protocol，IP 路由协议）**
  ：生成 Routes。
* **RIB（Routing Information Base，路由信息表）**
  ：存放 Routes。
* **LDP（Label Distribution Protocol，标签分发协议）**
  ：生成 Labels。
* **LIB（Label Information Base，标签信息表）**
  ：存放 Labels。

MPLS Data Plane 负责 IP Packets 的 Label 封装、Label 解封装、Label 转发、Routing 转发等。

* **FIB（Forwarding Information Base，转发信息表）**
  ：从 RIB 获取 Routes 来完成 IP Packet 转发。
* **LFIB（Label Forwarding Information Base，标签转发信息表）**
  ：从 LIB 获取 Labels 来完成 MPLS Packet 转发。

![](/assets/network-basic-route-mplss25.png)

  


### MPLS 的基本转发原理

传统的 IP Packets 路由决策，Router 首先需要识别 IP Header，再根据 dstIP 计算出归属的 FEC（Forwarding Equivalence Class，转发等价类）。在传统的采用最长匹配算法的 IP Routing 中，匹配到同一条 Route 的所有 IP Packets 就是一个 FEC，Router 在确定了 FEC 后即可完成最终的路由转发。

相对于 FEC，MPLS 提出了称为 LSH（Label Switching Hop，基于 Label 的转发）的新方法：

1. 当 IP Packets 首次进入 MPLS 网络时，首先还是会对 Packet 进行解包、分类、查找。然后，根据 MPLS 网络配置为不同的 IP Packets 分配对应的 Label。
2. 当 MPLS Packets 在 MPLS 网络中进行传输时，后面的所有路由决策都是基于 Label 和 NHLFE（Next Hop Label Forwarding Entry）来完成的。NHLFE 包含了 Label 和 Next Hop 的映射关系，以及需要对 Label 执行的操作，包括：替换（SWAP）、删除（POP）、添加（PUSH）。直到 IP Packets 流出 MPLS 网络为止。

这样带来了两个直观的好处：

1. Router 不需要解析 IP Header 了，MPLS Label 的解析由硬件芯片来完成。
2. 因为 MPLS Label 被设计为了 Integer 类型，可以达到 O\(1\) 的查找时间。

因此，MPLS 的 LSH 可以大大的减少路由决策的时间。

标签转发表的生成流程如下图所示：

1. IP 路由协议建立邻居，交互路由信息，生成 IP 路由表。
2. 标签交换协议从 IP 路由表中获取路由信息。IP 路由表中的路由前缀匹配了 FEC。
3. IP 路由表中激活的最优路由生成 IP 转发表。
4. 标签转换协议建立邻居，为 FEC 分配标签并发布给邻居，同时获取邻居发布的标签，生成标签转发表。

![](/assets/network-basic-route-mplss31.png)

# MPLS VPN 广域网专线

上文介绍了 MPLS 协议最初的目的是为了提高 L3 Router 的转发速度而设计的。与传统 IP 路由方式相比，MPLS 在数据转发时，只需要在 Network Edge Gateway 分析 IP Header，而不用在每一跳的 L3 Router 上都分析，因此节约了大量的计算时间。

然后在现代网络中，随着 ASIC 和 NP 等数通芯片技术的发展，路由查找速度已经不再是阻碍网络发展的瓶颈了，这使得 MPLS 在提高转发速度方面不再具备明显的优势。不过，MPLS 凭借其支持多层标签、转发平面面向连接等特性，使其在 VPN、TE（Traffic Engineering，流量工程）、QoS 等广域网专线的场景中仍然得到了广泛的应用。

MPLS VPN，即：基于 ISP 电信运营商 MPLS 广域网络之上构建的 VPN，也被称之为 MPLS 专线服务，本质是一种广域网 Underlay 专用线路租赁服务。运营商把专线租给用户，并承诺这条线路的 SLA（Service Level Agreement，服务等级协议，包括带宽、时延、抖动、丢包率等）。常用于企业多分支和 DCI（数据中心连接）场景。通过这条 MPLS 专线，跨域广域网的多个 Sites（站点）之间，逻辑上构建了一个企业内网。

根据具体应用场景的不同，MPLS VPN 还可以细分为 L2VPN 和 L3VPN 这两大类型。

### MPLS L2VPN

MPLS L2VPN 是一个 L2 over MPLS 网络，用于在多个 Sites 之间建立 L2 交换网络，支持实现多种不同的数据链路层，包括：ATM、FR、VLAN、Ethernet、PPP 等等。

另外，MPLS L2VPN 的技术实现种类有很多，例如：VLL、PWE3 和 VPLS 等。由于 MPLS L2VPN 中 PE 不参与用户流量的 L3 路由处理，因此 PE 的横向扩展性要比 L3VPN 要好得多，只与 PE 能连接的 VPN 用户数目相关。但是作为代价，L2VPN 的灵活性要差一些。

值得注意的是，随着 EVPN Control Plane 的兴起和发展，基于 VPLS Control Plane 的 MPLS L2VPN 已经逐渐被替代，所以这里只做简单的介绍。

### MPLS L3VPN

## MPLS L3VPN 是本文介绍的重点，也是更加被广泛应用的类型。之所以称之为 L3VPN 是因为其 Control Plane 基于 MP-BGP 动态路由协议来实现，而 Data Plane 仍然是 MPLS。 ![](/assets/network-basic-route-mplss32.png) MPLS L3VPN 工作原理

### 基本组网元素

在 MPLS L3VPN 的模型中，网络由 ISP 的骨干网与用户的各个 Sites 组成，组网方式灵活、可扩展性好，并能够方便地支持 MPLS QoS 和 MPLS TE，因此至今是一种主流方案。

经典的 MPLS L3VPN 网络架构主要由若干个 Sites 和 3 类网元组成：

1. **Site（用户站点）**
   ：表示一个本地的用户网络（例如：公司总部、分支机构），一个用户站点可以通过一条或多条链路连接到 ISP 的骨干网络。
2. **CE（Customer Edge，用户边缘路由器）**
   ：用户侧设备，是接入 ISP 的 DC-GW（IP 路由器，或是一台主机），与连接的一个或多个 PE 建立邻接关系。CE 不需要必须支持 MPLS 协议栈。
3. **PE（Provider Edge，骨干网边缘路由器）**
   ：接入层设备，ISP 的路由器，连接 P 和 CE，相当于 LER（标签边缘路由器）。在 MPLS 网络中，对 VPN 的所有处理都发生在 PE 上。
4. **P（Provider，骨干网路由器）**
   ：核心层设备，ISP 的路由器，连接 PE，不连接 CE，相当于 LSR（标签交换路由器）。只需要具备基本的数据面转发能力。

![](/assets/network-basic-route-mplss33.png)

### MP-BGP VPN-IPV4 Routes 类型

VPN-IPV4 Routes 是一种 BGP MP 扩展，通过扩展了 BGP NLRI（Network Layer Reachability Information，网络层可达信息）中的 VPNv4 地址族，用于支撑 MPLS L3VPN 的应用场景，具体而言，它具有下述的 3 个核心字段。

#### VPN-IPv4 字段

传统的 BGP-4 协议无法正确处理地址空间重叠的 VPN Routes，例如：VPN1 和 VPN2 都使用了 10.110.10.0/24 网段，并且各自都发布了一条 NextHop Local Routes，而 BGP-4 只能选择保留其中一条 Route，从而丢失了去往另一个 VPN 的路由。

为了解决地址空间重叠的问题，MPLS L3VPN 实现了互相隔离的 VPN Instances。每个 VPN Instance 都拥有专属的 VPN-IPv4 地址空间，而每个 VPN-IPv4 地址均由 RD（Route Distinguisher，路由区分符）和 IPv4 地址 2 大部分组成。即：在 IPv4 地址字段前面再增加了一个 8Byte 的 RD 字段，用来区分不同 VPN Instance 的 IPV4 地址。

如下图所示，VPN-IPv4 地址共有 12 Byte，包括了 8Byte 的 RD 和 4Byte 的 IPv4 地址前缀。

![](/assets/network-basic-route-mplss35.png)

#### RD（Route Distinguisher，路由区分符）字段

RD 的作用是添加到一个特定的 IPv4 前缀，使之成为全局唯一的 VPN-IPv4 前缀。

RD 的 Value 自身通常有 2 种组成方式：

1. **与 ASN（自治系统号）关联**
   ：RD 由 ASN 和一个任意数字组成；
2. **与 IP 地址关联**
   ：RD 是由 IP 地址和一个任意数字组成。

相对的，RD 有 3 种类型，通过 Type 字段区分：

1. **Type == 0**
   ：2Byte Administrator，4Byte Assigned number。格式为：
   `{16bits ASN}:{32bits Custom number}`
   ，例如：
   `100:1`
   。
2. **Type == 1**
   ：4Byte Administrator，2Byte Assigned number。格式为：
   `{32bits IPv4}:{16bits Custom number}`
   ，例如：
   `172.1.1.1:1`
   。
3. **Type == 2**
   ：4Byte Administrator，2Byte Assigned number。格式为：
   `32bits ASN}:{16bits Custom number}`
   ，其中的 ASN 最小值为 65536，例如：
   `65536:1`
   。

建议为 PE 上每个 VPN Instances 都配置专门的 RD，以保证到达同一哥 CE 的 Routes 都使用相同的 RD。而 RD 为 0 的 VPN-IPv4 地址，就相当于是全局唯一的 IPv4 地址了。

ISP 是运行独立划分 RD 的，但必须保证 RD 的全局唯一性，这样即使不同的 ISP 的 VPN 使用了重叠的 IPv4 地址，PE 也可以向各 VPN 发布不同的 Routes 了。

为保证 RD 的全局唯一性，建议不要将 Administrator 字段的值设置为私有的 ASN 或私有的 IP 地址。

#### RT（Route Target，路由目标）字段

MP-BGP 的 RT（Route Target，路由目标）扩展字段用于实现 PE 的 VPN Instance 的 Route Target Filter（目标路由过滤器）功能，可以控制 VPN Routes 的发布。

PE 上的 VPN Instance 有 2 类 RT 字段：

1. **Export Target**
   ：Local PE 发布 VPN-IPv4 Routes 到 Remote PE 之前，为这些 Routes 设置 Export Target 属性；
2. **Import Target**
   ：Local PE 在接收到 Remote PE 发布的 VPN-IPv4 Route 时，检查其 Export Target 属性，只有当 Export Target 与 Local PE 上的 VPN Instance 的 Import Target 属性匹配时，才会把这些 Routes 加入到相应的 VPN VRF 中。

PE 针对每个 VPN VRF 都可以配置一些 Export Target / Import Target Policies，规定了一个 VRF 可以接收携带何种 RT 属性的路由信息，向外发布路由时携带什么 RT 属性。

也就是说，VPN-IPV4 Route 所携带的 RT 属性决定了 VPN 的成员关系。RT 属性定义了一条 VPN-IPv4 Routes 可以被那些 PE 接收，或者说，定义了 PE 可以接收哪些 VPN-IPv4 Routes。

与 RD 类似，RT 也有 3 种格式：

1. `{16bits ASN}:{32bits Custom number}`
   ，例如：
   `100:1`
   。
2. `{32bits IPv4}:{16bits Custom number}`
   ，例如：
   `172.1.1.1:1`
   。
3. `{32bits ASN}:{16bits Custom number}`
   ，其中的 ASN 最小值为 65536。例如：
   `65536:1`
   。

### PE 设备支持 VPN-IPV4 Routes 类型

#### PE 的 VPN Instance

在 MPLS L3VPN 中，一个 VPN 表述一个用户（租户），不同 VPN 之间的路由隔离通过 VPN Instance 实现。

PE 为每个直接相连的 CE 建立并维护专门的 VPN Instance。VPN Instance 中包含对应 Site 的 VPN 成员关系和路由规则。如果一个 Site 中的用户同时属于多个 VPN，则该 Site 的 VPN Instance 中将包括所有这些 VPN 的信息。为保证 VPN 数据的独立性和安全性，PE 上每个 VPN Instance 都有相对独立的 VRF 和 LFIB。

具体来说，VPN Instance 中的信息包括：

1. LFIB（Label Forwarding Information Base，标签转发表）；
2. VRF（VPN Routing 
   &
    Forwarding，VPN 的路由转发表）；
3. 与 VPN Instance 绑定的 Interface；
4. VPN Instance 的管理信息，包括：RD（Route Distinguisher，路由区分符）、RT（Route Target，路由目标）路由过滤策略、成员接口列表等。

#### PE 的 VRF

PE Virtualization 是 VRF 之于 Router，类似 VLAN 之于 Switch，Trunk 之于 Ethernet Connection，VDOM 之于 Firewall，VM 之于 Server。VRF 的本质就是一种 Path Isolation（路径隔离）技术。

在 MPLS L3VPN 的 PE 中，维护了一张全局的路由表（公网路由表）。同时，还会维护着多张 VRF 表，VRF 是一种路由隔离和信息隔离的方式，使得每个 VRF 实例相互隔离，可以将 VRF 看作是一个 vPE 设备。

这些 VRF 是和 PE 上的一个或多个 VPN Instance 相对应，用于存放这些 VPN Instance 的 VPN Routers Information。通常的，一个 VRF 中只包含一个 VPN Instance 的 Routers。

通常的，一个 VRF 中包括了以下元素：

1. RIB（Routing Information Base），独立的 Routes Table。
2. FIB（Forwarding Info Base），独立的 Forwarding Table。
3. 一组归属于本 VRF 的 Interfaces 的集合。
4. 一组只用于本 VRF 的路由协议。

### VPN-IPV4 Routes 的交换方式

在基本的 MPLS L3VPN 组网中，VPN Routes 的发布主要涉及 CE 和 PE。PE 也只维护与它直接相连的 VPN Routes，不维护所有 VPN Routes。而 P 只维护骨干网 PE 的路由，不需要了解任何 VPN Routes。因此，MPLS L3VPN 网络具有良好的可扩展性。

VPN Routes 的发布过程包括 3 部分：

1. 本地 CE 到入口 PE。
2. 入口 PE 到出口 PE。
3. 出口 PE 到远端 CE。

完成这 3 部分后，Local CE 与 Remote CE 之间将建立可达路由。

#### Local CE 和 Ingress PE 之间的路由交换

CE 通常是一台普通的 Router，当 CE 与直接相连的 PE 建立了 Adjacency（邻接）关系后，CE 就把 Local Site 的 VPN Routers 发布给 PE，并从 PE 学到 Remote VPN 的 Routes。

CE 与 PE 之间交换 Routes，可以采用静态路由、或动态路由（e.g. RIP、OSPF、BGP、IS-IS）的方式。而采用静态路由方式的好处是可以降低因为 CE 管理不善等原因导致对骨干网 BGP 路由产生了震荡，影响了骨干网的稳定性。

Ingress PE 学习了 Local CE Site 的 VPN Routes 后，PE 会为这些 VPN Routes 增加 RD 和 RT 属性，形成标准的 VPN-IPv4 Routes。然后再发布到 Local CE 的 VPN Instance 中，也会通过 BGP 发布到其它 Egress PE。

#### Ingress PE 和 Egress PE 之间的路由交换

PE 只会维护与它直接相连的 VPN Routes，不会维护 ISP 网络中所有的 VPN Routes。

Ingress PE 通过 MP-BGP 把 VPN-IPv4 Routes 发布给 Egress PE。Egress PE 根据 VPN-IPv4 Routes 的 Export Target 属性与自己维护的 VPN Instance 的 Import Target 属性，决定是否将该 Routes 加入到 VPN Instance 的 Route Table 中。

#### Egress PE 和 Remote CE 之间的路由交换

Remote CE 有多种方式可以从 Egress PE 学习 VPN Routes，包括：静态路由、RIP、OSPF、IS-IS 和 E-BGP。与 Local CE 到 Egress PE 的路由信息交换方式相同。

### L3VPN 的报文转发流程

用户接入 MPLS L3VPN 的方式是为每个 Site 提供一个或多个 CE，再与 PE 连接。在 PE 上为这个 Site 配置 VRF，将连结 PE-CE 的物理接口、逻辑接口、甚至 L2TP/IPSec 隧道绑定到 VRF 上。

当在 MPLS 骨干网上传输 VPN 流量时，入口 PE 做为 Ingress LER，出口 PE 做为 Egress LER，P 路由器则做为 Transit LSR。

在基本 MPLS L3VPN 应用中（不包括跨域的情况），VPN 报文转发采用两层标签方式：

1. **第一层（外层）标签**
   ：在骨干网内部进行交换，指示从 Local PE 到 Remote PE 的一条 LSP（标签转发路径）。VPN 报文利用这层标签，可以沿 LSP 到达 Remte PE；
2. **第二层（内层）标签**
   ：在从 Remote PE 到达 CE 时使用，指示报文应被送到哪个 CE。

特殊情况下，属于同一个 VPN 的两个 CE 连接到同一个 PE，这种情况下只需要知道如何到达对端 CE。

以下图为例，说明 VPN 报文的转发过程：

1. Site1 发出一个目的地址为 1.1.1.2 的 IP 报文，由 CE1 将报文发送至 PE1。
2. PE1 根据报文到达的接口及目的地址查找 VPN Instance 中的 VRF 和 LFIB 表项，匹配后将报文转发出去，同时打上内层和外层两个标签。
3. MPLS 网络利用报文的外层标签，将报文传送到 PE2（报文在到达 PE2 前一跳时已经被剥离外层标签，仅含内层标签）。
4. PE2 根据内层标签和目的地址查找 VPN Instance 中的 VRF 和 LFIB 表项，确定报文的出接口，将报文转发至 CE2。
5. CE2 根据正常的 IP 转发过程将报文传送到目的地。

![](/assets/network-basic-route-mplss41.png)



## MPLS L3VPN 的经典组网方案

### 单域 L3VPN 场景

单域 MP-BGP MPLS VPN 在一个 AS 内运行，任何 VPN 的 Routes 都是只能在一个 AS 内按需扩散。

### ![](/assets/network-basic-route-mplss42.png)基本组网方案

最简单的情况下，一个 VPN 中的所有用户形成闭合用户群，相互之间能够进行流量转发，VPN 中的用户不能与任何本 VPN 以外的用户通信。

对于这种组网，需要为每个 VPN Instance 分配一个 Route Target（VPN Target），作为该 VPN Routes 的 Export Target / Import Target 属性值。并且，此 Route Target 通常是全局唯一的。

下图中，PE 上为 VPN1 分配的 Route Target 值为 100:1，为 VPN2 分配的 Route Target 值为 200:1。VPN1 的两个 Site 之间可以互访，VPN2 的两个 Site 之间也可以互访，但 VPN1 和 VPN2 的 Site 之间不能互访。

![](/assets/network-basic-route-mpss43.png)

#### Hub & Spoke 组网方案

如果希望在 VPN 中设置中心访问控制设备，其它用户的互访都通过中心访问控制设备进行，可以使用 Hub&Spoke 组网方案，从而实现中心设备对两端设备之间的互访进行监控和过滤等功能。

对于这种组网，需要设置两个 VPN Target，一个表示 Hub，另一个表示 Spoke。

各 Site 在 PE 上的 VPN Instance 的 Route Target 设置规则为：

1. **连接 Spoke 站点（Site1 和 Site2）的 Spoke-PE**
   ：Export Target 为 Spoke，Import Target 为 Hub；
2. **连接 Hub 站点（Site3）的 Hub-PE**
   ：Hub-PE 上需要使用两个接口或子接口，一个用于接收 Spoke-PE 发来的路由，其 VPN Instance 的 Import Target 为 Spoke；另一个用于向 Spoke-PE 发布路由，其 VPN Instance 的 Export Target 为 Hub。

下图中，Spoke 站点之间的通信通过 Hub 站点进行（图中箭头所示为 Site2 的路由向 Site1 的发布过程）：

1. Hub-PE 能够接收所有 Spoke-PE 发布的 VPN-IPv4 路由；
2. Hub-PE 发布的 VPN-IPv4 路由能够为所有 Spoke-PE 接收；
3. Hub-PE 将从 Spoke-PE 学到的路由发布给其他 Spoke-PE。因此，Spoke 站点之间可以通过 Hub 站点互访。
4. 任意 Spoke-PE 的 Import Target 属性不与其它 Spoke-PE 的 Export Target 属性相同。因此，任意两个 Spoke-PE 之间不直接发布 VPN-IPv4 路由，Spoke 站点之间不能直接互访。

![](/assets/network-basic-route-mplss45.png)

#### Extranet 组网方案

如果一个 VPN 用户希望提供部分本 VPN 的站点资源给非本 VPN 的用户访问，可以使用 Extranet 组网方案。

对于这种组网，如果某个 VPN 需要访问共享站点，则该 VPN 的 Export Target 必须包含在共享站点的 VPN Instance 的 Import Target 中，而其 Import Target 必须包含在共享站点 VPN Instance 的 Export Target 中。

下图中，VPN1 的 Site3 能够被 VPN1 和 VPN2 访问：

* PE3 能够接受 PE1 和 PE2 发布的 VPN-IPv4 路由；
* PE3 发布的 VPN-IPv4 路由能够为 PE1 和 PE2 接受；
* 基于以上两点，VPN1 的 Site1 和 Site3 之间能够互访，VPN2 的 Site2 和 VPN1 的 Site3 之间能够互访。

PE3 不把从 PE1 接收的 VPN-IPv4 路由发布给 PE2，也不把从 PE2 接收的 VPN-IPv4 路由发布给 PE1（IBGP 邻居学来的条目是不会再发送给别的 IBGP 邻居），因此，VPN1 的 Site1 和 VPN2 的 Site2 之间不能互访。

![](/assets/network-basic-route-mplss46.png)

### 跨域 L3VPN 场景

  
![](/assets/network-basic-route-mplss47.png)

如果同一个 VPN 的 2 个 Site 位于不同的 AS，意味着连接 VPN 的 PE 已经无法简单地建立 I-BGP 邻居关系、或是与 RR 建立邻居关系了。因此，需要一些手段通过建立 E-BGP 邻居关系来传递 VPNv4 Routes。

为了支持不同 AS 之间的 VPNv4 Routes 交换，就需要使用跨域（Inter-AS）的 MPLS VPN 架构了，以便可以穿过 AS 间的链路来发布路由前缀和标签信息。

MPLS 有三种不同的 LDP \(Label Distribution Protocol\) 部署选项，即 Option A、B 和 C。这些选项定义了如何在 MPLS 网络中分发标签和构建标签交换路径（LSP）。

#### Inter-Provider Backbones Option A

Option A 是一种简单的部署选项，每个 LSR（Label Switching Router）只分发到其相邻的下游 LSR 的标签。这种方式需要在网络中维护大量的标签交换路径，因此不适用于大型网络。

简单的说就是每个 AS 只在自己内部通过 MP-iBGP 宣告路由。需要手动的在跨域 VPN 的 ASBR 间通过专用的接口管理自己的 VPN Routes，也称为 VRF-to-VRF。

#### ![](/assets/network-basic-route-mplss48.png)Inter-Provider Backbones Option B

Option B 是一种折中的部署选项，每个 LSR 只与其相邻的下游 LSR 通信，但是在某些情况下会向网络中的其他 LSR 分发标签。这种方式可以在一定程度上减少标签和路径信息的数量，适用于中等规模的网络。

简单的说就是，处理在每个 AS 内部使用 MP-iBGP 宣告路由之外，不同的 AS 之间还会通过 ASBR PE 使用 MP-eBGP 宣告彼此 AS 的路由信息。

![](/assets/network-basic-route-mplss49.png)



#### Inter-Provider Backbones Option C

Option C 是一种复杂的部署选项，可以实现在整个网络中分布 Lables。每个 LSR 分别与网络中所有其他 LSR 通信，以构建完整的标签交换路径。这种方式需要在网络中维护大量的标签和路径信息，因此不适用于大型网络。

简单的说，就是整网都使用 MP-eBGP 宣告路由。

![](/assets/network-basic-route-mplss410.png)



