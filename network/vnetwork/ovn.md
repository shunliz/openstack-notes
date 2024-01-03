# OVN [https://www.ovn.org/en/](https://www.ovn.org/en/)

### 架构![](/assets/network-ovn-architecture.png)

* Openstack/CMS plugin 是 CMS 和 OVN 的接口，将CMS 的配置转化成 OVN 的格式写到 Nnorthbound DB 。

* Northbound DB 存储逻辑数据，与传统网络设备概念一致，比如 logical switch，logical router，ACL，logical port。

* ovn-northd 类似于一个集中式控制器，把Northbound DB 里面的数据翻译后写到 Southbound DB 。

* Southbound DB 保存的数据和 Northbound DB 语义完全不一样，主要包含 3 类数据，一是物理网络数据，比如 HV（hypervisor）的 IP 地址，HV 的 tunnel 封装格式；二是逻辑网络数据，比如报文如何在逻辑网络里面转发；三是物理网络和逻辑网络的绑定关系，比如逻辑端口关联到哪个 HV 上面。

* ovn-controller 是 OVN 里面的 agent，类似于 neutron 里面的 ovs-agent，运行在每个 HV 上面，北向，ovn-controller 会把物理网络的信息写到 Southbound DB，南向，把 Southbound DB 保存的数据转化成 Openflow flow 配到本地的 OVS table 里面，来实现报文的转发。

* ovs-vswitchd 和 ovsdb-server 是 OVS 的两个进程。

### 安装

apt-get update

apt-get -y install build-essential fakeroot

sudo apt-get install python-six openssl -y

sudo apt-get install openvswitch-switch openvswitch-common -y

sudo apt-get install ovn-central ovn-common ovn-host -y

sudo apt-get install ovn-host ovn-common -y

启动

/etc/init.d/openvswitch-switch start

/etc/init.d/ovn-central start

ovs-vsctl add-br br-int – set Bridge br-int fail-mode=secure

配置

ovn-nbctl set-connection ptcp:6641:172.18.22.197

ovn-sbctl set-connection ptcp:6642:172.18.22.197

ovs-vsctl set open . external-ids:ovn-remote=tcp:172.18.22.197:6642

ovs-vsctl set open . external-ids:ovn-encap-type=geneve

ovs-vsctl set open . external-ids:ovn-encap-ip=172.18.22.197

ovs-vsctl set open . external-ids:ovn-remote=tcp:172.18.22.197:6642

ovs-vsctl set open . external-ids:ovn-encap-type=geneve

ovs-vsctl set open . external-ids:ovn-encap-ip=172.18.22.198

## OVN Northbound DB {#OVN架构--高正伟-OVNNorthboundDB}

Northbound DB 是 OVN 和 CMS 之间的接口，Northbound DB 保存CMS 产生的数据，ovn-northd 监听数据库的内容变化，然后翻译并保存到 Southbound DB 里面。Northbound DB 里面主要有如下几张表：

* Logical\_Switch：每一行代表一个逻辑交换机，逻辑交换机有两种，一种是 overlay logical switches，对应于 neutron network，每创建一个 neutron network，networking-ovn 会在这张表里增加一行；另一种是 bridged logical switch，连接物理网络和逻辑网络，被 VTEP gateway 使用。Logical\_Switch 里面保存了它包含的 logical port（指向 Logical\_Port table）和应用在它上面的 ACL（指向 ACL table）。
* Logical\_Port：每一行代表一个逻辑端口，每创建一个 neutron port，networking-ovn 会在这张表里增加一行，每行保存的信息有端口的类型，比如 patch port，localnet port，端口的 IP 和 MAC 地址，端口的状态 UP/Down。
* ACL：每一行代表一个应用到逻辑交换机上的 ACL 规则，如果逻辑交换机上面的所有端口都没有配置 security group，那么这个逻辑交换机上不应用 ACL。每条 ACL 规则包含匹配的内容，方向，还有动作。
* Logical\_Router：每一行代表一个逻辑路由器，每创建一个 neutron router，networking-ovn 会在这张表里增加一行，每行保存了它包含的逻辑的路由器端口。
* Logical\_Router\_Port：每一行代表一个逻辑路由器端口，每创建一个 router interface，networking-ovn 会在这张表里加一行，它主要保存了路由器端口的 IP 和 MAC。

## OVN Southbound DB {#OVN架构--高正伟-OVNSouthboundDB}

Southbound DB 处在 OVN 架构的中心，它是 OVN 中非常重要的一部分，它跟 OVN 的其他组件都有交互。 Southbound DB 里面有如下几张表：

* Chassis：每一行表示一个 HV 或者 VTEP 网关，由 ovn-controller/ovn-controller-vtep 填写，包含 chassis 的名字和 chassis 支持的封装的配置（指向表 Encap），如果 chassis 是 VTEP 网关，VTEP 网关上和 OVN 关联的逻辑交换机也保存在这张表里。
* Encap：保存着 tunnel 的类型和 tunnel endpoint IP 地址。
* Logical\_Flow：每一行表示一个逻辑的流表，这张表是 ovn-northd 根据 Nourthbound DB 里面二三层拓扑信息和 ACL 信息转换而来的，ovn-controller 把这个表里面的流表转换成 OVS 流表，配到 HV 上的 OVS table。流表主要包含匹配的规则，匹配的方向，优先级，table ID 和执行的动作。  
* Multicast\_Group：每一行代表一个组 bZ播组，组播报文和广播报文的转发由这张表决定，它保存了组播组所属的 datapath，组播组包含的端口，还有代表 logical egress port 的 tunnel\_key。
* Datapath\_Binding：每一行代表一个 datapath 和物理网络的绑定关系，每个 logical switch 和 logical router 对应一行。它主要保存了 OVN 给 datapath 分配的代表 logical datapath identifier 的 tunnel\_key。
* Port\_Binding：每一行包含的内容主要有 logical port 的 MAC 和 IP 地址，端口类型，端口属于哪个 datapath binding，代表 logical input/output port identifier 的 tunnel\_key, 以及端口处在哪个 chassis。端口所处的 chassis 由 ovn-controller/ovn-controller 设置，其余的值由 ovn-northd 设置。

  表 Chassis 和表 Encap 包含的是物理网络的数据，表 Logical\_Flow 和表 Multicast\_Group 包含的是逻辑网络的数据，表 Datapath\_Binding 和表 Port\_Binding 包含的是逻辑网络和物理网络绑定关系的数据。

## OVN Chassis {#OVN架构--高正伟-OVNChassis}

Chassis 是 OVN 新增的概念，Chassis 是HV/VTEP 网关。Chassis 的信息保存在 Southbound DB 里面，由 ovn-controller/ovn-controller-vtep 来维护。以 ovn-controller 为例，当 ovn-controller 启动的时候，它去本地的数据库 Open\_vSwitch 表里面读取**external\_ids:system\_id**，**external\_ids:ovn-remote**，**external\_ids:ovn-encap-ip** 和**external\_ids:ovn-encap-type**的值，然后它把这些值写到 Southbound DB 里面的表 Chassis 和表 Encap 里面。**external\_ids:system\_id**表示 Chassis 名字。**external\_ids:ovn-remote**表示 Sounthbound DB 的 IP 地址。**external\_ids:ovn-encap-ip**表示 tunnel endpoint IP 地址，可以是 HV 的某个接口的 IP 地址。**external\_ids:ovn-encap-type**表示 tunnel 封装类型，可以是 VXLAN/Geneve/STT。**external\_ids:ovn-encap-ip**和**external\_ids:ovn-encap-type**是一对，每个 tunnel IP 地址对应一个 tunnel 封装类型，如果 HV 有多个接口可以建立 tunnel，可以在 ovn-controller 启动之前，把每对值填在 table Open\_vSwitch 里面。

## OVN tunnel {#OVN架构--高正伟-OVNtunnel}

OVN 支持的 tunnel 类型有三种，分别是 Geneve，STT 和 VXLAN。HV 与 HV 之间的流量，只能用 Geneve 和 STT 两种，HV 和 VTEP 网关之间的流量除了用 Geneve 和 STT 外，还能用 VXLAN，这是为了兼容硬件 VTEP 网关，因为大部分硬件 VTEP 网关只支持 VXLAN。虽然 VXLAN 是数据中心常用的 tunnel 技术，但是 VXLAN header 是固定的，只能传递一个 VNID（VXLAN network identifier），如果想在 tunnel 里面传递更多的信息，VXLAN 实现不了。所以 OVN 选择了 Geneve 和 STT，Geneve 的头部有个 option 字段，支持 TLV 格式，用户可以根据自己的需要进行扩展，而 STT 的头部可以传递 64-bit 的数据，比 VXLAN 的 24-bit 大很多。

OVN tunnel 封装时使用了三种数据：

* Logical datapath identifier（逻辑的数据通道标识符）：datapath 是 OVS 里面的概念，报文需要送到 datapath 进行处理，一个 datapath 对应一个 OVN 里面的逻辑交换机或者逻辑路由器，类似于 tunnel ID。这个标识符有 24-bit，由 ovn-northd 分配的，全局唯一，保存在 Southbound DB 里面的表 Datapath\_Binding 的列 tunnel\_key 里。
* Logical input port identifier（逻辑的入端口标识符）：进入 logical datapath 的端口标识符，15-bit 长，由 ovn-northd 分配的，在每个 datapath 里面唯一。它可用范围是 1-32767，0 预留给内部使用。保存在 Southbound DB 里面的表 Port\_Binding 的列 tunnel\_key 里。
* Logical output port identifier（逻辑的出端口标识符）：出 logical datapath 的端口标识符，16-bit 长，范围 0-32767 和 logical input port identifier 含义一样，范围 32768-65535 给组播组使用。对于每个 logical port，input port identifier 和 output port identifier 相同。

如果 tunnel 类型是 Geneve，Geneve header 里面的 VNI 字段填 logical datapath identifier，Option 字段填 logical input port identifier 和 logical output port identifier，TLV 的 class 为 0xffff，type 为 0，value 为 1-bit 0 + 15-bit logical input port identifier + 16-bit logical output port identifier。如果 tunnel 类型是 STT，上面三个值填在 Context ID 字段，格式为 9-bit 0 + 15-bit logical input port identifier + 16-bit logical output port identifier + 24-bit logical datapath identifier。OVS 的 tunnel 封装是由 Openflow 流表来做的，所以 ovn-controller 需要把这三个标识符写到本地 HV 的 Openflow flow table 里面，对于每个进入 br-int 的报文，都会有这三个属性，logical datapath identifier 和 logical input port identifier 在入口方向被赋值，分别存在 openflow metadata 字段和 Nicira 扩展寄存器 reg14 里面。报文经过 OVS 的 pipeline 处理后，如果需要从指定端口发出去，只需要把 Logical output port identifier 写在 Nicira 扩展寄存器 reg15 里面。

OVN tunnel 里面所携带的 logical input port identifier 和 logical output port identifier 可以提高流表的查找效率，OVS 流表可以通过这两个值来处理报文，不需要解析报文的字段。 OVN 里面的 tunnel 类型是由 HV 上面的 ovn-controller 来设置的，并不是由 CMS 指定的，并且 OVN 里面的 tunnel ID 又由 OVN 自己分配的，所以用 neutron 创建 network 时指定 tunnel 类型和 tunnel ID（比如 vnid）是无用的，OVN 不做处理。

## OVN中的信息流

### 配置数据

OVN中的配置数据从北向南流动。

CMS使用其OVN/CMS插件，通过北向数据库来将逻辑网络配置传递给ovn-northd。  
另外一边，ovn-northd将配置编译成一个较低级别的形式，并通过南向数据库将其传递给所有chassis。

### 状态信息

OVN中的状态信息从南向北流动。

OVN目前只提供几种形式的状态信息。

* 1、首先：ovn-northd填充北向Logical\_Switch\_Port表中的up列：  
  如果南向端口绑定表中的逻辑端口chassis列非空，则将其设为true，否则为false。  
  这让CMS可以检测VM的网络何时起来（come up）。

* 2、其次：OVN向CMS提供其配置实现的反馈，即CMS提供的配置是否已经生效。  
  该特性需要CMS参与序列号协议（sequence number protocol），其工作方式如下：

  * 当CMS更新北向数据库中的配置时，作为同一事务的一部分，它将增加`NB_Global`表中的`nb_cfg`列的值。（只有当CMS想知道配置何时实现时，这才是必要的。）
  * 当ovn-northd基于北向数据库的给定快照（snapshot）更新南向数据库时，作为同一事务的一部分，它将`nb_cfg`从北向数据库的`NB_Global`复制到南向数据库的`SB_Global`表中。（这样一来，监视两个数据库的观察者就可以确定南向数据库何时赶上北向数据库。）
  * 在ovn northd从southerbound数据库服务器收到其更改已提交的确认之后，它将`NB_Global`表中的`sb_cfg`更新为被已被推到下面的`nb_cfg`版本。（这样一来，CMS或另一个观察者可以确定南向数据库何时被挂起（caught up），而不用与南向数据库连接）。
  * 每个chassis上的ovn-controller控制器进程接收更新后的南向数据库和更新后的`nb_cfg`。该过程也会更新chassis的Open vSwitch实例上安装的物理流（physical flows）。当它从Open vSwitch收到物理流已更新的确认信息时，就会在南向数据库中更新自己的Chassis记录中的nb\_cfg。
  * ovn-northd 监视所有南向数据库中Chassis记录的nb\_cfg列。它跟踪所有记录中的最小值，并将其复制到北向
    `NB_Global`表的`hv_cfg`列中。（这样一来，CMS或另一个观察者就可以确定什么时候所有hypervisor都赶上了北向配置。）

## 逻辑网络（Logical Network）

逻辑网络（logical network）实现了与物理网络（physical network）一样的概念，不过他们通过通道\(tunnel\)或者其他封装手段，与物理网络隔离开来。这就让逻辑网络可以拥有单独的IP和其他地址空间，如此一来，便可以与物理网络所用的IP和地址空间无冲突地重叠。安排逻辑网络拓扑的时候，可以不用考虑它们运行所在的物理网络拓扑。

OVN里的逻辑网络概念包含:

* **逻辑交换机**
  （Logical switch）：以太网交换机的逻辑版本
* **逻辑路由器**
  （Logical router）：IP路由器的逻辑版本。逻辑交换机及路由器都可以接入复杂的拓扑里。
* **逻辑数据通路**
  （Logical datapath）：逻辑版本的OpenFlow交换机。逻辑交换机及路由器都以`逻辑数据通路`形式实现。
* **逻辑端口**  
  （Logical port）：代表了逻辑交换机和逻辑路由器之间的连接点。一些常见类型的逻辑端口有：

  * 逻辑端口（Logical port）:代表VIF。
  * 本地网络端口（Localnet port）：代表逻辑交换机与物理网络之间的连接点。  
    他们实现为OVS补丁端口（OVS patch port），架设在集成网桥和单独的Open vSwitch网桥之间，从而作为底层网络连接的端口。

  * 逻辑补丁端口（Logical patch port）：代表了逻辑交换机和逻辑路由器之间的连接点，  
    有时候还可作为对等逻辑路由器（peer logical router）连接点。每个这样的连接点都有一对逻辑补丁端口，每侧一个。

  * 本地端口端口（Localport port）：代表了逻辑路由器和VIF之间的本地连接点。  
    这些端口存在于每个机箱中（未绑定到任何特定的机箱），来自它们的流量永远不会通过隧道（tunnel）。  
    本地端口一般来说只生成目标指向本地目的地的通信，通常是对其接收到请求的响应。其中一个用例便是：`OpenStack Neutron`如何使用Localport端口为每个hyperviso上的VM提供元数据（metadata）服务。  
    元数据代理进程连接到每个主机（host）上的此端口，同一个网络中的所有VM都将通过相同的IP/MAC地址来访问它，

## VIF的生命周期

单独呈现表（Table）和其模式的话会很难理解。这里有一个例子。

虚拟机监控程序（hypervisor）上的VIF是一个虚拟网络接口，他们要么连接到VM上要么连接到容器上（容器直接在该hypervisor上运行）。这与在VM中运行的容器的接口不同:

本例中的步骤通常涉及到OVN和OVN Northbound database模式的详细信息。想了解这些数据库的完整信息，请分别参阅`ovn-sb`和`ovn-nb`。

* 1、当CMS管理员使用CMS用户界面或API创建新的VIF，并将其添加到交换机（由OVN作为逻辑交换机实现）时，VIF的生命周期开始。CMS更新自身配置，主要包括：将VIF唯一的持久标识符`vif-id`和以太网地址mac相关联。

* 2、CMS插件通过向`Logical_Switch_Port`表添加一行来更新OVN北向数据库以囊括新的VIF信息。新行之中，名称是`vif-id`，mac是mac，交换机指向OVN逻辑交换机的Logical\_Switch记录，其他列则被适当地初始化。

* 3、ovn-northd收到OVN北向数据库的更新。相应的，它做出响应，在南向数据库的`Logical_Flow`表的添加新行，以映射新的端口。比如：添加一个流来识别发往给新端口的MAC地址的数据包，并且更新传递广播和多播数据包的流以囊括新的端口。它还在“绑定”表中创建一条记录，并填充用于识别chassis的列之外的所有列。

* 4、每个hypervisor上的`ovn-controller`都会接收上一步ovn-northd在Logical\_Flow表所做的更新。但如果持有VIF的虚拟机关机，ovn-controller就无能为力了。例如，它不能发送数据包或从VIF接收数据包，因为VIF实际上并不存在了。

* 5、最终，用户启动拥有该VIF的VM。在VM启动的hypervisor上，hypervisor与Open vSwitch（IntegrationGuide.rst中所描述的）之间的集成，是通过将VIF添加到OVN集成网桥上实现的，此外还需要在`external_ids：iface-id`中存储`vif-id`，以指示该接口是新VIF的实例。

* 6、在启动VM的hypervisor上，ovn-controller在新的接口中注意到`external_ids：iface-id`。作为响应，在OVN南向数据库中，它将更新绑定表的chassis列中的相应行，这些行链接逻辑端口`external_ids：iface-id`到hypervisor。之后，ovn-controller更新本地虚拟机hypervisor的OpenFlow表，以便正确处理进出VIF的数据包。

* 7、一些CMS系统，包括OpenStack，只有在网络准备就绪的情况下才能完全启动虚拟机。  
  为了支持这个功能，ovn-northd一旦发现Binding表中的chassis列的某行更新了，则通过更新OVN北向数据库的`Logical_Switch_Port`表中的up列来向上标识这个变化，以表明VIF现在已经启动。如果使用此功能，CMS则可通过允许VM执行来继续进行后面的反应。

* 8、在VIF所在的每个hypervisor上，ovn-controller会觉察到绑定表中完全填充的行。这为ovn-controller提供了逻辑端口的物理位置，因此每个实例都会更新其交换机的OpenFlow流表（基于OVN数据Logical\_Flow表中的逻辑数据路径流），以便通过隧道来正确处理进出VIF的数据包。

* 9、最终，用户关闭持有该VIF的VM。在VM关闭的hypervisor上，VIF将从OVN集成网桥中删除。

* 10、在VM关闭的hypervisor上，ovn-controller发现到VIF已被删除。作为响应，它将删除逻辑端口绑定表中的Chassis列的内容。

* 11、在每个hypervisor上，ovn-controller都会发现绑定表行中空的Chassis列。  
  这意味着ovn-controller不再知道逻辑端口的物理位置，因此每个实例都会更新其OpenFlow表以反映这一点。

* 12、最终，当VIF（或其整个VM）不再被任何人需要时，管理员使用CMS用户界面或API删除VIF。CMS更新自己的配置。

* 13、CMS插件通过删除Logical\_Switch\_Port表中的相关行来从OVN北向数据库中删除VIF。

* 14、ovn-northd收到OVN北向数据库的更新，然后相应地更新OVN南向数据库，  
  方法是：从OVN南向数据库的Logical\_Flow表和绑定表中删除或更新与已销毁的VIF相关的行。

* 15、在每个hypervisor上，ovn-controller接收在上一步中的Logical\_Flow表的更新内容，并更新OpenFlow表。

  不过可能没有太多要做，因为VIF已经变得无法访问，它在上一步中就从绑定表中删除了。

### VM内部容器接口的生命周期

OVN通过将写在`OVN_NB`数据库中的信息，转换成每个hypervisor中的OpenFlow流表来提供虚拟网络抽象。

如果想要为多租户提供安全的虚拟网络连接，那么OVN控制器便应该是唯一可以修改Open vSwitch中的流的实体时候。当Open vSwitch集成网桥驻留在虚拟机管理程序中时，虚拟机内运行的租户工作负载（ tenant workloads ）是无法对Open vSwitch流进行任何更改的。

如果基础架构provider相信容器内的应用程序不会中断和修改Open vSwitch流，则可以在hypervisors中运行容器。当容器在虚拟机中运行时也是如此，此时需要有一个驻留在同一个VM中并由OVN控制器控制的Open vSwitch集成网桥。  
对于上述两种情况，工作流程与上一节（“VIF的生命周期”）中的示例所解释的相同。

本节讨论在虚拟机中创建容器并将Open vSwitch集成网桥驻留在虚拟机管理程序中时，容器接口（CIF）的生命周期。  
在这种情况下，即使容器应用程序发生故障，其他租户也不会受到影响，因为运行在VM内的容器无法修改Open vSwitch集成网桥中的流。

在虚拟机内创建多个容器时，会有多个CIF关联。为了便于OVN支持虚拟网络抽象，与这些CIF关联的网络流量需要到达hypervisor中运行的Open vSwitch集成网桥。OVN还应能够区分来自不同CIF的网络流量。

### 区分CIP网络流量的方法

有两种方法可以区分CIF的网络流量：

#### 1:1

第一种方法是为每个CIF都提供一个VIF（1:1）：

这意味着hypervisor中可能有很多网络设备。由于管理所有VIF会造成额外的CPU消耗，这会使OVS变慢。这也意味着在VM中创建容器的实体也应该能够在hypervisor中创建相应的VIF。

#### 1:2

第二种方法是为所有CIF只提供一个VIF（1:n）。然后，OVN可以通过每个数据包中写入的标签来区分来自不同CIF的网络流量。  
OVN使用这种机制并使用VLAN作为标记机制。

* 1、CIF的生命周期从虚拟机内部创建容器开始，该容器可以是由创建虚拟机的同一CMS创建，或拥有该虚拟机的租户创建，  
  甚至可以是由与最初创建虚拟机的CMS不同的容器编制系统所创建。  
  无论创建容器的实体是谁，它需要知道与虚拟机的网络接口关联的`vif-id`，容器接口的网络流量将通过该网络接口来流通。  
  创建容器接口的实体也需要在该虚拟机内部选择一个未使用的VLAN。

* 2、创建容器的实体（直接或间接通过CMS管理底层基础结构）通过向`Logical_Switch_Port`表添加一行来更新OVN南向数据库以囊括新的CIF。  
  在新行中，`name`是任何唯一的标识符，`parent_name`是CIF网络流量预计要经过的VM的vif-id，`tag`是标识该CIF网络流量的VLAN标记。

* 3、ovn-northd收到OVN北向数据库的更新。反过来，它通过添加相应行到OVN南向数据库的`Logical_Flow`表来反映新的端口，  
  并通过在Binding表中创建一个新行\(填充除了标识列以外的所有列\)，来相应地更新OVN南向数据库的chassis。

* 4、在每个hypervisor上，ovn-controller订阅绑定表中的更改。  
  当由ovn-northd创建的新行中包含Binding表的parent\_port列中的值时，拥有与`external_ids：iface-id`中的vif-id相同值的ovn集成网桥上的ovn-controller,将会更新本地hypervisor的OpenFlow流表，这样来自具有特定VLAN标记的VIF的数据包便可以得到正确的处理。之后，它会更新绑定表的chassis列以反映物理位置。

* 5、底层网络准备就绪后，便只能在容器内启动应用程序。  
  为了支持这个功能，ovn-northd在绑定表中通知更新的chassis列，并更新OVN Northbound数据库的Logical\_Switch\_Port表中的up列，以表示CIF现在已经启动。负责启动容器应用程序的实体查询到该值并启动应用程序。

* 6、最后，创建并启动容器的实体，会停止容器。实体则通过CMS（或直接）删除Logical\_Switch\_Port表中的行。

* 7、ovn-northd接收OVN Northbound更新，并相应地更新OVN Southbound数据库，  
  方法是从OVN Southbound数据库Logical\_Flow表中删改与已经销毁的CIF相关联的行。同时也会删除该CIF绑定表中的行。

* 8、在每个hypervisor中，ovn-controller接收在上一步中`Logical_Flow`表的更新。ovn-controller会更新本地的`OpenFlow`表以反映更新。

## 数据包的物理处理生命周期

本节介绍数据包如何通过OVN从一个虚拟机或容器传输到另一个。

这个描述着重于数据包的物理处理。有关数据包逻辑生命周期的描述，请参考ovn-sb中的Logical\_Flow表。

为了清楚起见，本节提到了一些数据和元数据字段，总结如下：

### 涉及的数据和元数据字段

* **隧道密钥**。当OVN在Geneve或其他隧道中封装数据包时，会附加额外的数据，以使接收的OVN实例正确处理。  
  取决于特定的封装形式，会采取不同的形式，但在每种情况下，我们都将其称为“隧道密钥”。  
  请参阅文末的“隧道封装”以了解详细信息。

* **逻辑数据路径字段**。表示数据包正在被处理的逻辑数据路径的字段。  
  OVN使用`OpenFlow 1.1+`简单地（且容易混淆地）调用“元数据”来存储logical datapath字段。  
  （该字段作为隧道密钥的一部分通过隧道进行传递。）

* **逻辑输入端口字段**。表示数据包从哪个逻辑端口进入logical datapath的字段。  
  OVN将其存储在Open vSwitch扩展寄存器14中。Geneve和STT隧道将这个字段作为隧道密钥的一部分传递。  
  虽然VXLAN隧道没有明确地带有逻辑输入端口，OVN只使用VXLAN与网关进行通信，从OVN的角度来看，  
  只有一个逻辑端口，因此OVN可以在输入到OVN时，将`逻辑输入端口`字段设置为该逻辑输入的逻辑管道。

* **逻辑输出端口字段**。表示数据包将离开logical datapath的逻辑端口的字段。在逻辑入口管道的开始，它被初始化为0。  
  OVN将其存储在Open vSwitch扩展寄存器15中。Geneve和STT隧道将这个字段作为隧道密钥的一部分传递。  
  VXLAN隧道不传输逻辑输出端口字段，由于VXLAN隧道不在隧道密钥中携带逻辑输出端口字段，  
  所以当OVN管理程序从VXLAN隧道接收到数据包时，将数据包重新提交给流表8以确定输出端口。  
  当数据包到达流表32时，通过检查数据包从VXLAN隧道到达时所设置的MLF\_RCV\_FROM\_VXLAN标志，  
  将这些数据包重新提交到表33以供本地传送。

* **逻辑端口的conntrack区域字段**。表示逻辑端口的连接跟踪区域的字段。  
  该值只在本地有意义，跨chassis之间是没有意义的。在逻辑入口管道的开始，它被初始化为0。  
  OVN将其存储在Open vSwitch扩展寄存器13中。

* **路由器的conntrack区域字段**。表示路由器的连接跟踪区域的字段。  
  该值只在本地有意义，跨chassis之间是没有意义的。OVN在Open vSwitch扩展寄存器11中存储`DNATting`的区域信息，  
  在Open vSwitch扩展寄存器12中存储`SNATing`的区域信息。

* **逻辑流标志**。逻辑标志旨在处理保持流表之间的上下文以便确定后续表中的哪些规则匹配。  
  该值只在本地有意义，跨chassis之间是没有意义的。OVN将逻辑标志存储在Open vSwitch扩展寄存器10中。

* **VLAN ID**。VLAN ID用作OVN和嵌套在VM内的容器之间的接口（有关更多信息，请参阅上面VM内容器接口的生命周期）。

### 详细的生命周期

首先，hypervisor上的VM或容器通过连接到OVN集成网桥的端口上,来发送数据包。详细的生命周期如下：

* 1、OpenFlow流表0执行物理到逻辑的转换。它匹配数据包的入口（ingress）端口。
  其通过设置`逻辑数据路径`字段以标识数据包正在穿越的逻辑数据路径，并设置`逻辑输入端口`字段来标识入口端口，从而实现用逻辑元数据标注数据包。然后重新提交到表8进入逻辑入口管道。

`源自嵌套在虚拟机内的容器的数据包有稍微不同的方式处理。可以根据VIF特定的VLAN ID来区分始发容器，因此物理到逻辑的转换流程将在VLAN ID字段上需要匹配，且在VLAN header剥离时也会进行匹配。在这一步之后，OVN像处理其他数据包一样处理来自容器的数据包。`

流表0还处理从其他chassis到达的数据包。它通过入口端口（这是一个隧道）将它们与其他数据包区分开来。与刚刚进入OVN流水线的数据包一样，这些操作使用逻辑数据路径和逻辑入口端元数据来注释这些数据包。另外，设置逻辑输出端口字段的操作也是可用的，因为在OVN中，隧道发生在逻辑输出端口已知之后。这三个信息是从隧道封装元数据中获取的（请参阅隧道封装了解编码细节）。然后操作重新提交到表33以进入逻辑出口管道。

* 2、OpenFlow流表8至31执行OVN Southbound数据库中的Logical\_Flow表的逻辑入口管道。  
  这些流表完全用于逻辑端口和逻辑数据路径等逻辑概念的表示。ovn-controller的一大部分工作就是把它们转换成等效的OpenFlow（尤其是，它翻译数据库表：把Logical\_Flow表0到表23翻译为成为OpenFlow流表8到31）。

  每个逻辑流都映射到一个或多个OpenFlow流。一个实际的数据包通常只匹配其中一个，尽管在某些情况下它可以匹配多个这样的数据流（这不成问题，因为它们动作都一样）。ovn-controller使用逻辑流的`UUID`的前32位作为OpenFlow流的cookie。（这不一定是唯一的，因为逻辑流的UUID的前32位不一定是唯一的。）

  一些逻辑流程可以映射到Open vSwitch``连接匹配(`conjunctive match)``扩展（参见ovs-field部分文档）。流连接操作使用的`OpenFlow cookie`为0，因为它们可以对应多个逻辑流。关于OpenFlow流的连接匹配包含`conj_id`上的匹配。

  如果某个逻辑流无法在该hypervisor上使用，则某些逻辑流可能无法在给定的hypervisor中的OpenFlow流表中表示。例如，如果逻辑交换机中没有VIF驻留在给定的hypervisor上，并且该hypervisor上的其他逻辑交换机不可访问（例如，从hypervisor上的VIF开始的逻辑交换机和路由器的一系列跳跃），那么逻辑流就可能不可以在那里表示。

  大多数OVN操作在OpenFlow中具有相当明显的实现（具有OVS扩展），例如，`next`实现为`resubmit，field = constant`; 至于`set_field`，下面有一些更详细地描述：

  * `output`  
    通过将数据包重新提交给表32来实现。如果pipeline执行多于一个的输出动作，则将每个动作单独地重新提交给表32。  
    这可以用于将数据包的多个副本发送到多个端口。（如果数据包在输出操作之间未被修改，并且某些拷贝被指定给相同的hypervisor，那么使用逻辑多播输出端口将节省hypervisor之间的带宽。）

  * `get_arp(P, A)`

  * `get_nd(P, A)`  
    通过将参数存储在OpenFlow字段中来实现，然后重新提交到表66，ovn-controller使用OVN南向数据库中的`MAC_Binding`表生成的此66流表。如果表66中存在匹配项，则其会将绑定的MAC存储在以太网目的地地址字段中。

  > OpenFlow操作保存并恢复用于上述参数的OpenFlow字段，OVN操作不必知道这个临时用途。

  * `put_arp(P, A, E)`
  * `put_nd(P, A, E)`
    通过将参数存储在OpenFlow字段中实现，通过ovn-controller来更新MAC\_Binding表。OpenFlow操作保存并恢复用于上述参数的OpenFlow字段，OVN操作不必知道这个临时用途。

* 3、OpenFlow流表32至47在逻辑入口管道中执行输出动作。  
  具体来说就是表32处理到远程hypervisors的数据包，表33处理到hypervisors的数据包，表34检查其逻辑入口和出口端口相同的数据包是否应该被丢弃。

  逻辑patch端口是一个特殊情况。逻辑patch端口没有物理位置，并且有效地驻留在每个hypervisor上。因此，用于输出到本地hypervisor上的端口的流表33也自然地将输出实现到单播逻辑patch端口。但是，将相同的逻辑应用到作为逻辑多播组一部分的逻辑patch端口会产生数据包重复，因为在多播组中包含逻辑端口的每个hypervisor也会将数据包输出到逻辑patch端口。因此，多播组执行表32中的逻辑patch端口的输出。

  表32中的每个流匹配逻辑输出端口上的单播或多播逻辑端口，其中包括远程hypervisor上的逻辑端口。每个流的操作实现就是发送一个数据包到它匹配的端口。对于远程hypervisor中的单播逻辑输出端口，操作会将隧道密钥设置为正确的值，  
  然后将隧道端口上的数据包发送到正确的远端hypervisor。（当远端hypervisor收到数据包时，表0会将其识别为隧道数据包，并将其传递到表33中。）对于多播逻辑输出端口，会将数据包的一个副本发送到每个远程hypervisor，就像单播目的地一样。如果多播组包括本地hypervisor上的一个或多个逻辑端口，则其动作也重新提交给表33. 表32还包括：

  * 根据标志`MLF_RCV_FROM_VXLAN`匹配从VXLAN隧道接收到的数据包的高优先级的规则，然后将这些数据包重新提交给表33进行本地传递。从VXLAN隧道收到的数据包由于缺少隧道密钥中的逻辑输出端口字段而到达此处，因此需要将这些数据包提交到表8以确定输出端口。

  * 根据逻辑输入端口匹配从localport类型的端口接收到的数据包并将这些数据包重新提交到表33以进行本地传送的较高优先级规则。每个 hypervisor都存在localport类型的端口，根据定义，它们的流量不应该通过隧道出去。

  * 如果没有其他匹配，则重新提交到流表33的备用流。

  对于驻留在本地而不是远程的逻辑端口，表33中的流表类似于表32中的流表。对于本地hypervisor中的单播逻辑输出端口，操作只是重新提交给表34。对于在本地hypervisor中包括一个或多个逻辑端口的多播输出端口，对于每个这样的逻辑端口`P`，操作将逻辑输出端口改变为`P`，然后重新提交到表34。

  一个特殊情况是，当数据路径上存在localnet端口时，会通过切换到localnet端口来连接远程端口。在这种情况下，不是在表32中增加一个到达远程端口的流，而是在表33中增加一个流以将逻辑输出端口切换到localnet端口，并重新提交到表33，好像它被单播到本地hypervisor的逻辑端口上一样。

  表34匹配并丢弃逻辑输入和输出端口相同并且`MLF_ALLOW_LOOPBACK`标志未被设置的数据包。它重新提交其他数据包到表40。

* 4、OpenFlow表40至63执行OVN Southbound数据库中的Logical\_Flow表的逻辑出口流程。出口管道可以在分组交付之前执行验证的最后阶段。最终，它可以执行一个输出动作，ovn-controller通过重新提交到表64来实现。pipeline从不执行输出的数据包被有效地丢弃（虽然它可能已经穿过物理网络的隧道传送了）。

> 出口管道不能改变逻辑输出端口或进行进一步的隧道封装。

* 5、当设置`MLF_ALLOW_LOOPBACK`时，表64绕过OpenFlow环回\(loopback \)。  
  逻辑环回在表34中被处理，但是OpenFlow默认也防止环回到OpenFlow入口端口。因此，当`MLF_ALLOW_LOOPBACK`被设置时，OpenFlow表64保存OpenFlow入口端口，将其设置为0，重新提交到表65以获得逻辑到物理转换，然后恢复OpenFlow入口端口，有效地禁用OpenFlow回送防止。当`MLF_ALLOW_LOOPBACK`未被设置时，表64流程仅仅重新提交到表65。

* 6、OpenFlow表65执行与表0相反的逻辑到物理\(logical-to-physica\)翻译。它匹配分组的逻辑输出端口。  
  其将数据包输出到代表该逻辑端口的OVN集成网桥端口。如果逻辑出口端口是一个与VM嵌套的容器，那么在发送数据包之前，会加上具有适当`VLAN ID`的`VLAN标头`再进行发送。

## 逻辑路由器和逻辑patch端口

通常，逻辑路由器和逻辑patch端口不都具有物理位置，并且实际上是驻留在hypervisor的。逻辑路由器和这些逻辑路由器后面的逻辑交换机之间的逻辑patch端口就是这种情况，VM（和VIF）连接到这些逻辑交换机。

考虑这样的情况：一个虚拟机或容器发送数据包到位于不同子网上的另一个VM或容器。数据包将按照前面“数据包的物理生命周期”部分所述，使用表示发送者所连接的逻辑交换机的logical datapath，遍历表0至65。在表32中，数据包本地重新提交到hypervisor上的表33的fallback flow。在这种情况下，从表0到表65的所有处理都在数据包发送者所在的hypervisor上进行。

当数据包到达表65时，逻辑出口端口是逻辑patch端口。表65中的实现取决于OVS版本，虽然观察到的行为意图是相同的：

* 在OVS版本2.6和更低版本中，表65输出到代表逻辑patch端口的OVS的patch端口。  
  在OVS patch端口的peer中，分组重新进入OpenFlow流表，标识其逻辑数据路径和逻辑输入端口都基于OVS patch端口的OpenFlow端口号。

* 在OVS 2.7及更高版本中，数据包被克隆并直接重新提交到入口管道中的第一个OpenFlow流表，将逻辑入口端口设置为对等逻辑patch端口，并使用对等逻辑patch端口的logical datapath（其实表示的是逻辑路由器）。

包重新进入入口流水线（ingress pipeline）以便再次遍历表8至65，这次使用表示逻辑路由器的逻辑数据路径。

处理过程继续，如前一部分“数据包的物理生命周期”部分所述。当数据包到达表65时，逻辑输出端口将再次成为逻辑patch端口。  
使用与上述相同的方式，这个逻辑patch端口将使得数据包被重新提交给OpenFlow表8至65，这次使用目标VM或容器所连接的逻辑交换机的逻辑数据路径。

该分组第三次也是最后一次遍历表8至65。如果目标VM或容器驻留在远程hypervisor中，则表32将在隧道端口上将该分组从发送者的hypervisor发送到远程hypervisor。最后，表65将直接将数据包输出到目标VM或容器。

以下部分描述了两个例外情况：逻辑路由器或逻辑patch端口与物理位置相关联。

### 网关路由器

网关路由器是绑定到物理位置的逻辑路由器。这包括逻辑路由器的所有逻辑patch端口以及逻辑交换机上的所有对等逻辑patch端口。  
在OVN Southbound数据库中，这些逻辑patch端口的Port\_Binding条目使用`l3gateway`类型而不是patch类型，以便区分这些逻辑patch端口绑定到chassis。

当hypervisor在代表逻辑交换机的逻辑数据路径上处理数据包，并且逻辑出口端口是表示与网关路由器连接的`l3gateway`时，该包将匹配表32中的流表，通过隧道端口将数据包发送给网关路由器所在的chassis。表32中的处理过程与VIF相同。

网关路由器通常用于分布式逻辑路由器和物理网络之间。分布式逻辑路由器及其后面的逻辑交换机（虚拟机和容器附加到的逻辑交换机）实际上驻留在每个hypervisor中。分布式路由器和网关路由器通过另一个逻辑交换机连接，有时也称为连接逻辑交换机。另一方面，网关路由器连接到另一个具有连接到物理网络的localnet端口的逻辑交换机。

当使用网关路由器时，DNAT和SNAT规则与网关路由器相关联，网关路由器提供可处理一对多SNAT（又名IP伪装）的中央位置。

### 分布式网关端口

分布式网关端口是逻辑路由器patch端口，可以将分布式逻辑路由器和具有本地网络端口的逻辑交换机连接起来。

分布式网关端口的主要设计目标，是尽可能多的在VM或容器所在的hypervisor上本地处理流量。只要有可能，就应该在该虚拟机或容器的hypervisor上，完全处理从该虚拟机或容器到外部世界的数据包，最终遍历该hypervisor上到物理网络的所有localnet端口实例。  
只要有可能，从外部到虚拟机或容器的数据包，就应该通过物理网络直接导向虚拟机或容器的hypervisor，数据包将通过localnet端口进入集成网桥。

为了允许上面段落中描述的分组分布式处理，分布式网关端口应该是有效地驻留在每个hypervisor上的逻辑patch端口，而不是绑定到特定chassis的l3gateway端口。但是，与分布式网关端口相关的流量通常需要与物理位置相关联，原因如下：

* localnet端口所连接的物理网络通常使用L2学习。任何通过分布式网关端口使用的以太网地址都必须限制在一个物理位置，这样上游的L2学习就不会混淆了。从分布式网关端口向具有特定以太网地址的localnet端口发出的流量，必须在特定chassis上发送出分布式网关端口的一个特定实例。  
  具有特定以太网地址的localnet端口（或来自与localnet端口相同的逻辑交换机上的VIF）的流量，必须定向到该特定chassis上的逻辑交换机的chassis端口实例。由于L2学习的含义，分布式网关端口的以太网地址和IP地址需要限制在一个物理位置。为此，用户必须指定一个与分布式网关端口关联的chassis。请注意，使用其他以太网地址和IP地址（例如，一对一NAT）穿过分布式网关端口的流量不限于此chassis。

  对ARP和ND请求的回复必须限制在一个单一的物理位置，回复中的以太网地址位于该位置。这包括分布式网关端口的IP地址的ARP和ND回复，这些回复仅限于用户与分布式网关端口关联的chassis。

* 为了支持一对多SNAT（又名IP伪装），其中跨多个chassis分布的多个逻辑IP地址被映射到单个外部IP地址，这将有必要以集中的方式处理特定chassis上的某些逻辑路由器。由于SNAT外部IP地址通常是分布式网关端口IP地址，且为了简单起见，使用与分布式网关端口相关联的相同chassis。

ovn-northd文档中描述了对特定chassis的流量限制的详细信息。

虽然分布式网关端口的大多数依赖于物理位置的地方，都可以通过将某些流限制到特定chassis来处理，但还是需要一个附加的机制。  
当数据包离开入口管道，且逻辑出口端口是分布式网关端口时，需要表32中两个不同的动作集中的一个：

* 如果可以在发送者的hypervisor上本地处理该分组（例如，一对一的NAT通信量），则该分组应该以正常的方式重新提交到表33，用于分布式逻辑patch端口。

* 但是，如果数据包需要在与分布式网关端口关联的chassis（例如，一对多SNAT流量或非NAT流量）上处理，则表32必须将该数据包在隧道端口上发送到该chassis。

为了触发第二组动作，已经添加了`chassisredirect`类型的南向数据库的Port\_Binding表。将逻辑出口端口设为`chassisredirect`类型的逻辑端口只是一种方式，表明虽然该数据包是指向分布式网关端口的，但需要将其重定向到不同的chassis。  
在表32中，具有该逻辑输出端口的分组被发送到特定的chassis，与表32将逻辑输出端口是VIF或者类型为13的网关端口的分组指向不同的chassis的方式相同。一旦分组到达该chassis，表33将逻辑出口端口重置为代表分布式网关端口的值。对于每个分布式网关端口，除了表示分布式网关端口的分布式逻辑patch端口外，还有一个`chassisredirect`类型的端口。

### 分布式网关端口的高可用性

OVN允许您为分布式网关端口指定chassis的优先级列表。这是通过将多个`Gateway_Chassis`行与`OVN_Northbound`数据库中的`Logical_Router_Port`关联来完成的。当为网关指定了多个chassis时，所有可能向该网关发送数据包的chassis都将启用通道上的BFD，这些隧道通往所有配置的网关chassis。当前网关的主chassis是当前最高优先级的网关chassis，该chassis根据BFD状态被当做主chassis。

## 连接到逻辑路由器的多个LocalNet逻辑交换机 {#9连接到逻辑路由器的多个localnet逻辑交换机}

可以有多个逻辑交换机，每个交换机都有一个本地网络端口（代表物理网络）连接到逻辑路由器，其中一个localnet逻辑交换机可以通过分布式网关端口提供外部连接，其余的localnet逻辑交换机在物理网络中使用VLAN标记。需要为所有这些localnet网络在chassis上正确配置`ovn网桥映射`（ovn-bridge-mappings）。

### 东西向路由 {#91东西向路由}

这些带有localnet VLAN标记的逻辑交换机之间的东西路由工作方式，与普通逻辑交换机几乎相同。当虚拟机发送这样的数据包时，则：

* 1、它首先进入源localnet逻辑交换机数据路径的入口（ingress）管道，然后进入出口\(egress\)管道。然后，它通过源chassis中的逻辑路由器端口进入逻辑路由器数据路径\(datapath\)的入口管道。

* 2、进行路由决策（Routing decision）。

* 3、数据包从路由器数据路径（datapath）进入目的地localnet逻辑交换机数据路径的入口管道，然后出口管道，并通过localnet端口从集成（integration）网桥再到提供者（provider）网桥（属于目的地逻辑交换机）。

* 4、目标chassis通过localnet端口接收数据包并将其发送到集成网桥（integration bridge）。数据包（packet）进入目的地localnet逻辑交换机的入口管道，然后离开管道，最后被送到目的地VM端口。

### 外部流量 {#92外部流量}

当虚拟机发送外部流量（需要NATting），并且承载虚拟机的chassis没有分布式网关端口时，会发生以下情况：

* 1、数据包首先进入源LocalNet逻辑交换机数据路径（datapath）的入口管道，然后离开管道。然后，它通过源chassis中的逻辑路由器端口进入逻辑路由器数据路径的入口管道。

* 2、进行路由决策。由于网关路由器或分布式网关端口不在源chassis中，流量通过隧道端口重定向到网关机箱。

* 3、网关chassis通过隧道端口接收数据包，数据包进入逻辑路由器数据路径的出口管道。NAT规则将在这里被应用。然后，数据包进入提供外部连接的localnet逻辑交换机数据路径的入口管道，接着离开管道，最后通过提供外部连接的逻辑交换机的localnet端口离开。

尽管这样做有效，但当从计算chassis发送到网关chassis时，VM通信量将被隧道化。为了使其正常工作，必须降低localnet逻辑交换机的MTU，以考虑隧道封装

### 连接到逻辑路由器的带有`localnet VLAN`标记的逻辑交换机的集中路由

为了克服上一节中描述的隧道封装\(tunnel encapsulation\)问题，OVN支持为带有`localnet VLAN`标记的逻辑交换机，启用集中路由的选项。

CMS可以配置选项：将所有连接到相应逻辑交换机（带有`localnet vlan`标记）logical\_router\_port的reside-on-redirect-chassis选项设为true ，这会导致网关chassis（承载分布式网关端口）处理这些网络的所有路由，使其集中化。它将回复对逻辑路由器端口IP的ARP请求。

如果逻辑路由器没有连接到提供外部连接的localnet逻辑交换机的分布式网关端口，则OVN将忽略此选项。当VM发送需要路由的东西向流量时，会发生以下情况：

* 1、包首先进入入口管道，然后进入源localnet逻辑交换机数据路径的出口管道，并通过源localnet逻辑交换机的localnet端口发送出去（而不是发送到路由器管道）。

* 2、网关chassis通过源localnet逻辑交换机的localnet端口接收数据包并将其发送到集成网桥。然后，数据包进入入口\(ingress\)管道，接着离开源localnet逻辑交换机数据路径的管道，并进入逻辑路由器数据路径的入口管道。

* 3、进行路由决策（Routing decision）。

* 4、数据包从路由器数据路径（datapath）进入目的地localnet逻辑交换机数据路径的入口管道，然后离开管道。接着，它通过localnet端口离开集成桥到提供者（provider）桥（属于目标逻辑交换机）。

* 5、目标chassis通过localnet端口接收数据包并将其发送到集成网桥。数据包进入目的地localnet逻辑交换机的入口管道，然后离开管道，最后到达目的地VM端口。

当vm发送需要NATting的外部流量时，会发生以下情况：

* 1、 包首先进入入口管道，然后进入源localnet逻辑交换机数据路径的出口管道，并通过源localnet逻辑交换机的localnet端口发送出去（而不是发送到路由器管道）。

* 2、网关机箱通过源localnet逻辑交换机的localnet端口接收数据包并将其发送到集成网桥。然后，数据包进入入口管道，之后离开源localnet逻辑交换机数据路径的管道，并进入逻辑路由器数据路径的入口管道。

* 3、进行路由决策（Routing decision）并应用NAT规则

* 4、数据包从路由器数据路径，进入提供外部连接的localnet逻辑交换机数据路径的入口管道，然后从出口管道流出。最后，它通过localnet端口离开集成桥到提供者桥（属于提供外部连接的逻辑交换机）。

对于反向外部流量，将发生以下情况：

* 1、网关机箱从提供外部连接的逻辑交换机的localnet端口接收数据包。然后，数据包进入localnet逻辑交换机（提供外部连接）的入口管道，接着离开管道。之后，数据包进入逻辑路由器数据路径的入口管道。

* 2、 逻辑路由器数据路径的入口管道应用unNATting规则。然后，数据包进入源localnet逻辑交换机的入口管道，接着离开管道。  
  由于源vm不在网关chassis中，因此数据包通过源逻辑交换机的localnet端口发送。

* 3、源chassis通过localnet端口接收数据包并将其发送到集成网桥。数据包进入源localnet逻辑交换机的入口管道，然后离开管道，最后被送到源VM端口。

### VTEP网关的生命周期

网关其实是一种chassis，用于在逻辑网络的OVN管理部分和物理VLAN之间转发流量，将基于隧道的逻辑网络扩展到物理网络。

以下步骤通常涉及OVN和VTEP数据库模式的详细信息。请分别参阅ovn-sb，ovn-nb和vtep，了解关于这些数据库的详细信息。

* 1、VTEP网关的生命周期始于管理员将VTEP网关注册为VTEP数据库中的`Physical_Switch`表项。连接到此VTEP数据库的`ovn-controller-vtep`将识别新的VTEP网关，并在OVN\_Southbound数据库中为其创建新的Chassis表条目。

* 2、然后，管理员可以创建一个新的`Logical_Switch`表项，并将VTEP网关端口上的特定vlan绑定到任意VTEP逻辑交换机。  
  一旦VTEP逻辑交换机绑定到VTEP网关，ovn-controller-vtep将检测到它，并将其名称添加到OVN\_Southbound数据库中chassis表的vtep\_logical\_switches列。

> 请注意，VTEP逻辑交换机的tunnel\_key列在创建时未被填充。当相应的vtep逻辑交换机绑定到OVN逻辑网络时，`ovn-controller-vtep`才将设置列内容。

* 3、现在，管理员可以使用CMS将VTEP逻辑交换机添加到OVN逻辑网络。为此，CMS必须首先在OVN\_Northbound数据库中创建一个新的Logical\_Switch\_Port表项。然后，该条目的类型列必须设置为“vtep”。接下来，还必须指定options列中的vtep-logical-switch和vtep-physical-switch密钥，因为多个VTEP网关可以连接到同一个VTEP逻辑交换机。

* 4、OVN\_Northbound数据库中新创建的逻辑端口及其配置将作为新的Port\_Binding表项传递给OVN\_Southbound数据库。  
  ovn-controller-vtep将识别更改内容并将逻辑端口绑定到相应的VTEP网关chassis。

> 禁止将同一个VTEP逻辑交换机绑定到不同的OVN逻辑网络，否则会在日志中产生警告。

* 5、除绑定到VTEP网关chassis外，ovn-controller-vtep还会将VTEP逻辑交换机的tunnel\_key列，更新为绑定OVN逻辑网络的相应Datapath\_Binding表项的tunnel\_key。

* 6、接下来，ovn-controller-vtep将对OVN\_Northbound数据库中的Port\_Binding中的配置更改作出反应，并更新VTEP数据库中的`Ucast_Macs_Remote`表。这使得VTEP网关可以了解从哪里转发来自扩展外部网络的单播流量。

* 7、最终，当管理员从VTEP数据库注销VTEP网关时，VTEP网关的生命周期结束。ovn-controller-vtep将识别该事件并删除OVN\_Southbound数据库中的所有相关配置（chassis表条目和端口绑定）。

* 8、当ovn-controller-vtep终止时，将清除OVN\_Southbound数据库和VTEP数据库中的所有相关配置，包括所有注册的VTEP网关及其端口绑定的Chassis表条目，以及所有Ucast\_Macs\_Remote表项和Logical\_Switch隧道密钥。

## 12 安全

### 12.1、针对南向数据库的基于角色的访问控制

为了提供额外的安全性以防止OVN chassis受到攻击，从而防止流氓软件对南向数据库状态进行任意修改，从而中断OVN网络，  
从而有了针对南向数据库的基于角色的访问控制策略（请参阅ovsdb-server部分以获取更多详细信息）。

基于角色的访问控制（RBAC）的实现需要在OVSDB模式中添加两个表：`RBAC_Role`表，该表由角色名称进行索引，并将可以对给定角色进行修改的各个表的名称映射到权限表中的单个行，其中包含该角色的详细权限信息，权限表本身由包含以下信息的行组成：

* Table Name  
  关联表的名称。本栏存在主要是为了帮助人们阅读本表内容。

* 认证（Auth Criteria）  
  一组包含列名称的字符串（或者是列的column:key对，这些列包含了string:string映射关系）。至少一个列或列的内容：要修改，插入或删除的行中的键值必须等于尝试作用于该行的客户端的ID,要删改的行中，至少有一个column:key值或者列的内容，必须和尝试作用于该行的客户端的ID相同，以便授权检查通过。如果授权标准为空，则禁用授权检查，并且该角色扥所有客户端将被视为已授权。

* 插入/删除（Insert/Delete）  
  行插入/删除权限\(布尔值\)，指示相关表是否允许增删行。 如果为true，则授权客户端增删行。

* 可更新的列（Updatable Columns）一组字符串，他们包含了可能由授权客户端更新或变异的列或 column:key对。只有在客户的授权检查通过，并且所有要修改的列都包含在这组可修改的列中时，才允许修改行中的列。

  OVN南向数据库的RBAC配置由ovn-northd维护。启用RBAC时，只允许对Chassis / Encap / Port\_Binding和MAC\_Binding表进行修改，并且重新分配如下：

  * Chassis

    * 授权：客户端ID必须与chassis 名称匹配。
    * 插入/删除：允许授权的行插入和删除。
    * 更新：授权时可以修改列`nb_cfg`，`external_ids`，`encaps`和`vtep_logical_switches`。

  * Encap授权

    * 授权：客户端ID必须与chassis 名称匹配。
    * 插入/删除：允许授权的行插入和删除。
    * 更新：列类型，选项和IP可以修改。

  * Port\_Binding

    * 授权：禁用（所有客户端都被认为是授权的\)。
      未来改进可能会添加列（或external\_ids的key），以控制哪个chassis允许绑定每个端口。
    * 插入/删除：不允许行插入/删除（ovn-northd维护此表中的行）
    * 更新：只允许对chassis列进行修改。

  * MAC\_Binding

    * 授权：禁用（所有客户被认为是授权的）
    * 插入/删除：允许行插入/删除。
    * 更新：`logical_port`，`ip`，`mac`和`datapath`列可以由ovn-controller修改。

要为ovn-controller连接到南向数据库启用RBAC，需要以下步骤：

* 1、为证书CN字段设置为chassis名称的每个chassis创建SSL证书

  例如，对于带有`external-ids：system-id = chassis-1`的chassis，通过命令`ovs-pki -B 1024 -u req + sign chassis-1 switch`。

* 2、当连接到南向数据库时配置每个ovn-controller使用SSL。

* 例如通过`ovs-vsctl set open . external-ids:ovn-remote=ssl:x.x.x.x:6642`。

* 3、使用“ovn-controller”角色配置南向数据库SSL远程。

  例如通过`ovn-sbctl set-connection role=ovn-controller pssl:6642`。

### 12.2、使用IPsec加密隧道通信

OVN隧道流量需流经物理路由器和交换机，而这些物理设备可能不受信任（公共网络中的设备），或者可能受到危害。对隧道流量启用加密可以防止对流量数据进行监视和操作。

隧道流量是用`IPsec`加密的。CMS设置北向`NB_Global`表中的`IPsec`列，以启用或禁用IPsec加密。如果ipsec为true，则所有OVN隧道都将加密。如果ipsec为false，则不会加密OVN隧道。

当CMS更新北向NB\_Global表中的ipsec列时，ovn-northd将该值复制到南向SB\_Global表中的ipsec列。每个chassis中的ovn-controller监视南向数据库，并相应地设置OVS隧道接口的选项。OVS隧道接口选项由`ovs-monitor-ipsec`守护程序监视，该守护程序通过配置IKE守护程序来启动ipsec连接。

Chassis使用证书相互验证。如果隧道中的另一端提供由受信任的CA签名的证书，并且公用名（CN）与预期的Chassis名匹配，则验证成功。

基于角色的访问控制（RBAC）中使用的SSL证书可以在IPsec中使用。也可以使用`ovs-pki`创建不同的证书。证书要求是x.509 version 3，并且将`CN`字段和`SubjectAltName`字段设置为Chassis名称。

在启用IPsec之前，需要在每个Chassis中安装CA证书、Chassis证书和私钥。  
请参阅`ovs vswitchd.conf.db`来设置基于CA的IPsec身份验证。

## 13 设计决策

### 13.1、隧道（tunnel）封装

OVN标注从一个hypervisor发送到另一个的逻辑网络包，这些包包含以下三个元数据，他们以encapsulation-specific的方式编码：

* 1、**24位逻辑数据路径标识符**，来自OVN Southbound数据库Datapath\_Binding表中的隧道密钥列。
* 2、**15位逻辑入口端口标识符**。`ID 0`被保留在OVN内部供内部使用。可以将ID 1到32767（包括端点）分配给逻辑端口（请参阅OVN Southbound Port\_Binding表中的tunnel\_key列）。
* 3、**16位逻辑出口端口标识符**。ID 0到32767与逻辑入口端口具有相同的含义。可以将32768到65535的ID分配给logical多播组（请参阅OVN Southbound Multicast\_Group表中的tunnel\_key列）。

对于hypervisor到hypervisor的流量，OVN仅支持Geneve和STT封装，原因如下：

* 1、只有STT和Geneve支持OVN使用的大量元数据（每个数据包超过32位）（如上所述）。
* 2、STT和Geneve使用随机化的UDP或TCP源端口，允许在底层使用ECMP的环境中，在多个路径之间进行高效分发。
* 3、NIC支持对STT和Geneve封装和解封装。

由于其灵活性，hypervisor之间的首选封装方案是`Geneve`。对于Geneve封装，OVN在`Geneve VNI`中传输逻辑数据路径标识符。  
OVN将类别为`0x0102`，类型为`0x80`的TLV中的逻辑入口端口和逻辑出口端口从MSB传输到LSB，其编码为32位，如下所示：

```
1
15
16

+---+------------+-----------+
|rsv|ingress port|egress port|
+---+------------+-----------+

0
```

不支持Geneve的网卡环境，因为性能的原因，可能更偏向STT封装。对于STT封装，OVN将STT 64位隧道ID中所有三个逻辑元数据编码如下，从MSB到LSB分别是：

```
9
15
16
24

+--------+------------+-----------+--------+
|reserved|ingress port|egress port|datapath|
+--------+------------+-----------+--------+

0
```

为了连接到网关，除了Geneve和STT之外，OVN还支持`VXLAN`，因为在top-of-rack \(ToR\)交换机上，只有VXLAN是普遍支持的。、  
目前，网关具有与由VTEP模式定义的能力匹配的特征集，因此元数据的比特数是必需的。将来，不支持封装大量元数据的网关可能会继续减少功能集。

### VTEP 网关 {#OVN架构--高正伟-VTEP网关}

Neutron 子项目networking-l2gw 支持L2Gateway，仅支持连接物理网络的 VLAN 到逻辑网络的 VXLAN.

![](/assets/adfasfsafasdfsadfewqr2311414324.png)

OVN 可以通过 VTEP 网关把物理网络和逻辑网络连接起来，VTEP 网关可以是 TOR（Top of Rack）switch，目前很多硬件厂商都支持，比如 Arista，Juniper，HP 等等；也可以是软件做的逻辑 switch，OVS 社区就做了一个简单的VTEP 模拟器。VTEP网关需要遵守 VTEP OVSDB schema，它里面定义了 VTEP 网关需要支持的数据表项和内容，VTEP 通过 OVSDB 协议与 OVN 通信，通信的流程 OVN 也有相关标准，VTEP 上需要一个 ovn-controller-vtep 来做 ovn-controller 所做的事情。VTEP 网关和 HV 之间常用 VXLAN 封装技术。

如图上图所示，PH1 和 PH2 是连在物理网络里面的 2 个服务器，VM1 到 VM3 是 OVN 里面的 3 个虚拟机，通过 VTEP 网关把物理网络和逻辑网络连接起来，从逻辑上看，PH1，PH2，VM1-VM3 就像在一个物理网络里面，彼此之间可以互通。

VTEP OVSDB schema 里面定义了三层的表项，但是目前没有硬件厂商支持，VTEP 模拟器也不支持，所以 VTEP 网络只支持二层的功能，也就是说只能连接物理网络的 VLAN 到逻辑网络的 VXLAN，如果 VTEP 上不同 VLAN 之间要做路由，需要 OVN 里面的路由器来做。

### 



