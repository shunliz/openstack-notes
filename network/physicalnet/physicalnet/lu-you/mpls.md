## Segment Routing

MPLS 需要在原有的 IGP 动态路由协议之上叠加 LDP（Label Distribution Protocol，标签分发协议）协议，由 LDP 协议来基于 IGP Route 生成 MPLS Label，并自动建立 IGP Route 信息到 MPLS Label 信息之间的映射关系。

但由于 LDP 协议不具备状态信息维护功能，所以 MPLS 标签转发与 IP 路由转发一样是无连接的、尽力而为的。同时，也因为 LDP 协议不具有 TE（流量工程）特性，所以为了解决 TE 可管理性方面的需求，所以还需要叠加额外的 RSVP（Resource Reservation Protocl-flow Engineering，基于流量工程扩展的资源预留协议）控制面协议。

相比于传统的、逐跳转发的转发方式，RSVP-TE 最大的优势在于其可以收集整网的拓扑和链路状态信息，并可以根据业务的需要灵活地选择流量的转发路径。如同我们开车使用的导航软件一般，它在计算行驶路线前需要收集道路信息并知晓当前路况，然后基于你的要求选择一条合适的路线。

然而 RSVP 信令非常复杂，同时还得维护庞大的链路信息，因此也存在信息交互效率低下，扩展困难等诸多难题。为了应对这些挑战，Segment Routing（SR）技术应运而生。

![](/assets/network-basic-route-mpls1.png)1977 年，在 Car A. Sunshine 发表了论文《Source routing in computer networks》中第一次提出了 Source Routing（源路由）的概念。直到 2013 年，由 Cisco 提出的 Segment Routing MPLS 方案才真正的实现了这种基于 Source Routing 理念进行设计的数据转发技术。

所谓 Source Routing，即：在数据发出的源头，就把路由路径确定下来，从而实现转发路径的可管理性。在 Segment Routing 网络中，将数据传输链路划分为了多个不同的 Segment（分段），并在 Ingress Node（路径起始节点）给数据报文插入 Segment List，以此来控制转发路径的选择。如下图所示。

1. 整个 SR 网络的拓扑被分成了多个不同的 Segments，并用 SID（Segment Identifier）来唯一标识。![](/assets/network-basic-route-mpls2.png)

2. 在 Ingress Node 给数据报文插入 Segment List（SID 的有序排列），本质是一个可控的转发路径信息。所有中间节点需要按照报文中携带的 Segment List 进行转发。![](/assets/network-basic-route-mpls3.png)

综上，一个典型的 Segment Routing 网络如下图所示：

1. **Segment**
   ：任意两个 SR Nodes（SR 路由器）之间的一段 Network Path（网络路径）。整个 SR 网络的拓扑被分成了多个不同的 Segments，并用 SID（Segment Identifier）来唯一标识。
2. **Segment  ID（SID）**
   ：Segment 的唯一标识。
3. * 在 SR-MPLS 中，每个 Segment 都被分配一个 MPLS Label 作为 SID；
   * 在 SRv6 中，每个 Segment 都使用一个特殊的 IPv6 地址作为 SID。
4. **Segment List（SID List）**
   ：作为存储 SIDs 的有序列表，通过对 SID 进行有序排列，就可以得到一条完整的 IP 报文转发路径。然后在路径的源头节点（Ingress Node）统一插入到 IP 报文中，SR 的中间节点只需要按照 IP 报文中携带的 Segment List 进行转发即可，并不感知和维护路径状态。
5. **SR Domain**
   ：一个区域中的 SR Nodes 的集合。

![](/assets/network-basic-route-mpls4.png)

同时，在 SR 技术设计之初，就已经考虑在 MPLS 和 IPv6 双 Data Plane 支持。

![](/assets/network-basic-route-mpls5.png)

在这里插入图片描述

* **SR-MPLS**
  ：在不改变 MPLS 基本转发行为的前提下，通过 MPLS 标签标识 Segment。
  ![](/assets/netowrk-basic-route-mpsl6.png)
* **SRv6**
  ：通过在 IPv6 Header 中增加 SRH 扩展字段，使用 SRH 头部中的 IPv6 地址标识 Segment，实现了基于 IPv6 的标签转发，从而完全舍弃了 MPLS。IPv6 独特的报文结构，可以与 SR 网络的运行原理完美搭配。
  ![](/assets/network-basic-route-mpls7.png)

## SR-MPLS

SR-MPLS 的 Data Plane 依旧是 MPLS，但 Control Plane 则使用扩展了 SR 属性字段的 IGP 协议来代替了 LDP 协议。并实现了基于源地址标签转发的 RSVP 集中控制，把所有的路径信息，下发给每个 SR Node，以此代替了 RSVP-TE 协议需要逐个节点进行特殊配置下发的方式。

![](/assets/network-basic-route-mpls8.png)

### SR-MPLS 的基本转发原理

![](/assets/network-basic-route-mpls9.png)SR-MPLS 将网络中的 Node （目标节点）和 Adjacency（邻接节点）定义为一个个 Segments，并为每个 Segment 分配唯一的 SID。通过 Node SID 和 Adjacency SID 组成的 Segment List 来描述一条 E2E 转发路径。然后由 Ingress Node 将 Segment List 封装到 Packet 的 MPLS Label Stack 中，最终按照 MPLS Data Plane 的转发机制逐跳转发。

而用于在 Node 和 Adjacency 中分配各自的 Segment ID 的 Control Plane 则由 IGP 协议来充当，支持 IS-IS、OSPF、I-BGP 等路由协议类型。其中：

* 每个 IGP Node Segment 对应的 Node SID 都是由电信运营商通过手工预先配置的，然后再由 IGP 对外进行宣告和收敛，全局可见，全局有效。
* 而 IGP Adjacency Segment 对应的 Adjacency SID 是一个 Local Segment，只在局部具有意义，仅仅用于标识 IGP 网络中的某个邻接关系。Adjacency SID 同样适用 IGP 协议对外宣告和收敛，全局可见，但仅本地有效。

![](/assets/network-basic-route-mpls10.png)

### SR-MPLS BE 转发原理

SR-MPLS BE（Best Effort）的实现基于 Node SID 结合 IGP 使用的 SPF 算法来计算出最短转发路径。例如：NodeZ 连接了 Destination Network（目标网络），该 Network 的 Node SID 是 68。通过 IGP 收敛之后，整个 IGP Domain 内所有的 Nodes 从 NodeZ 学习到 Destination Network 的 Node SID，之后所有的 Nodes 都会使用 SPF 算法得出一条这个 Destination Network 的最短路径。

![](/assets/network-basic-route-mpls11.png)

基于 SR-MPLS BE 技术建立的转发路径实际是一种 LSP（标签转发路径），只是这种 LSP 不存在 Tunnel  接口，称为 SR LSP。

SR LSP 的创建过程和数据转发与 MPLS LSP 类似，而 SR LSP 数据转发和 MPLS LSP 完全相同，包含标签栈压入（Push）、标签栈交换（Swap）和标签弹出（Pop）等动作，也可以支持倒数第二跳弹出特性 PHP（Penultimate Hop Popping），MPLS QoS 等特性。

![](/assets/network-basic-route-mpls13.png)



SR LSP 建立关键步骤：

1. **手工配置**
   ：Router 上配置 Node SID 和 SRGB，通过 IGP 报文宣告收敛。
2. **标签分配**
   ：各 Router 根据解析收到的 IGP 报文和自己的 SRGB 计算 Label 值，公式是：标签值 = 自己 SRGB 起始值 + Node SID 值；再根据下一跳节点发布的 SRGB 计算出标签值，公式是：出标签值 = 下一跳 SRGB 起始值 + Node SID 值。
3. **路径计算**
   ：各 Router 根据 IGP 收集到的拓扑，使用相同的 SPF 算法，计算标签转发路径，生成转发表项。

![](/assets/network-basic-route-mpls14.png)

### SR-MPLS TE 转发原理

SR-MPLS TE 是一种新型的 TE 隧道技术，称为 SR-MPLS TE Tunnel，支持 MPLS TE 隧道的相关属性，同时支持使用 BFD 检测故障。

![](/assets/network-basic-route-mpls15.png)

SR-MPLS TE 隧道可以通过 SDN 自动建立，也可以手工建立。通常由 SDN Controller 来构建。Controller 通过 BGP-LS 来收集网络信息，并根据 Policy（业务的需求）来计算出一条显示的路径。步骤如下：

1. **手工配置**
   ：Router 上配置 IGP SR，生成链路拓扑和标签信息。
2. **拓扑和标签信息上报**
   ：由 BGP-LS 上报给 SDN Controller。
3. **链路生成**
   ：由 PCEP 完成标签转发路径计算。
4. **信息下发**
   ：隧道属性配置和 LSP 信息分别由 NETCONF 和 PCEP 下发。
5. **隧道创建**
   ：PE 根据隧道属性和 LSP 信息自动创建 SR-TE 隧道。

SR-MPLS TE （Traffic Engineering） 有 2 种实现方式：

1. **基于 Adjacency Segment**：Ingress Node 指定严格的显式路径（Strict Explicit）。这种方式可以集中进行路径调整和流量调优，因此可以更好的配合实现 SDN。

![](/assets/network-basic-route-mpls21.png)

2. **基于 Adjacency Segment + Node Segment**

：显式路径与最短路径相结合，称作松散路径（Loose Explicit）。

![](/assets/network-basic-route-mpls22.png)

## SR-MPLS 在数据中心场景中的应用

值得注意的是，相较于越来越流行的 IPv6 技术，MPLS 技术这种通过插入 Label 来 Byapass 掉 IP 转发的方式所带来的直接后果是它丧失了 IP 技术的普适性，需要网络设备支持标签转发功能。基于这一技术原理，在某种程度上限定了 SR-MPLS 只能作为运营商专属的网络技术，一般只在运营商 IP 骨干网或者新建的城域网内采用，在数据中心和企业网中基本没有部署。

但随着可编程网络技术和主机虚拟网络技术的发展（e.g. DPDK、P4、DPU），解决了硬件层面限制，使得 MPLS 和 SR-MPLS 技术在数据中心场景中的应用探索也逐渐多了起来。下文中主要概括性的介绍相关的研究和应用场景。

### MPLS over UDP

MPLS over UDP 协议被应用于数据中心 Overlay 网络。

![](/assets/network-basic-route-mpls23.png)

### SR-MPLS over UDP

SR-MPLS over UDP 协议被应用于跨域多个数据中心的 Overlay 网络中。

![](/assets/network-basic-route-mpsl24.png)主要的 RFC 如下：

* SR-MPLS over IP（Draft）：https://datatracker.ietf.org/doc/html/draft-xu-mpls-sr-over-ip
* https://support.huawei.com/enterprise/en/doc/EDOC1100125786/4e8bf78b/introduction-of-segment-routing-mpls

SR-MPLS over UDP 的 Use Cases 如下：

* Segment Routing in the Data Center![](/assets/network-basic-route-mpls31.png)

* Routing Between Data Centers![](/assets/network-basic-route-mpls32.png)

* Tunneling SR Across a Non-SR Core![](/assets/network-basic-route-mpls33.png)

* SR in a Mixed Mode IP Network![](/assets/network-basic-route-mpls34.png)

### 基于 UDP srcPort 的负载均衡

通过将 MPLSoUDP 或 SR-MPLSoUDP 中的 UDP srcPort 设置为对 Original Raw Packet 执行 HASH 操作的结果，可以在多个不同的层面带来更好的负载均衡支持。

* **网络层面**
  ：当 Gateway 设备处理 MPLSoUDP Tunnel 时，可以基于 UDP srcPort 来实施 ECMP 负载均衡。

![](/assets/network-basic-route-mpls41.png)

  


* **主机 NIC 层面**
  ：在实施了 NICs Bond 的场景中，可以基于 UDP srcPort 来实施 Bond mode 负载均衡。

![](/assets/network-basic-route-mpls42.png)

  


* **主机 CPU 层面**
  ：在 DPDK 场景中，可以基于 UDP srcPort 来实现 CPU Cores 的负载均衡。

![](/assets/network-basic-route-mpls44.png)

  


