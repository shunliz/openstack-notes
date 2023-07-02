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
2. **Segment  ID（SID）**
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

![](/assets/network-basic-route-mpls9.png)

