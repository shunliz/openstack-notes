# OVN

### 架构![](/assets/network-ovn-architecture.png)

* Openstack/CMS plugin 是 CMS 和 OVN 的接口，将CMS 的配置转化成 OVN 的格式写到 Nnorthbound DB 。

* Northbound DB 存储逻辑数据，与传统网络设备概念一致，比如 logical switch，logical router，ACL，logical port。

* ovn-northd 类似于一个集中式控制器，把Northbound DB 里面的数据翻译后写到 Southbound DB 。

* Southbound DB 保存的数据和 Northbound DB 语义完全不一样，主要包含 3 类数据，一是物理网络数据，比如 HV（hypervisor）的 IP 地址，HV 的 tunnel 封装格式；二是逻辑网络数据，比如报文如何在逻辑网络里面转发；三是物理网络和逻辑网络的绑定关系，比如逻辑端口关联到哪个 HV 上面。

* ovn-controller 是 OVN 里面的 agent，类似于 neutron 里面的 ovs-agent，运行在每个 HV 上面，北向，ovn-controller 会把物理网络的信息写到 Southbound DB，南向，把 Southbound DB 保存的数据转化成 Openflow flow 配到本地的 OVS table 里面，来实现报文的转发。

* ovs-vswitchd 和 ovsdb-server 是 OVS 的两个进程。

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

### VTEP 网关 {#OVN架构--高正伟-VTEP网关}

```
  Neutron 子项目networking-l2gw 支持L2Gateway，仅支持连接物理网络的 VLAN 到逻辑网络的 VXLAN.
```

![](/assets/adfasfsafasdfsadfewqr2311414324.png)

OVN 可以通过 VTEP 网关把物理网络和逻辑网络连接起来，VTEP 网关可以是 TOR（Top of Rack）switch，目前很多硬件厂商都支持，比如 Arista，Juniper，HP 等等；也可以是软件做的逻辑 switch，OVS 社区就做了一个简单的VTEP 模拟器。VTEP网关需要遵守 VTEP OVSDB schema，它里面定义了 VTEP 网关需要支持的数据表项和内容，VTEP 通过 OVSDB 协议与 OVN 通信，通信的流程 OVN 也有相关标准，VTEP 上需要一个 ovn-controller-vtep 来做 ovn-controller 所做的事情。VTEP 网关和 HV 之间常用 VXLAN 封装技术。

如图上图所示，PH1 和 PH2 是连在物理网络里面的 2 个服务器，VM1 到 VM3 是 OVN 里面的 3 个虚拟机，通过 VTEP 网关把物理网络和逻辑网络连接起来，从逻辑上看，PH1，PH2，VM1-VM3 就像在一个物理网络里面，彼此之间可以互通。

VTEP OVSDB schema 里面定义了三层的表项，但是目前没有硬件厂商支持，VTEP 模拟器也不支持，所以 VTEP 网络只支持二层的功能，也就是说只能连接物理网络的 VLAN 到逻辑网络的 VXLAN，如果 VTEP 上不同 VLAN 之间要做路由，需要 OVN 里面的路由器来做。

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

