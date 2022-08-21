# OpenvSwitch架构

## OVS整体架构

![](/assets/network-virtualnetwork-ovs1.png)

ovs-vswitchd：守护程序，实现交换功能，和Linux内核兼容模块一起，实现基于流的交换flow-based switching。

ovsdb-server：轻量级的数据库服务，主要保存了整个OVS的配置信息，包括接口啊，交换内容，VLAN啊等等。ovs-vswitchd会根据数据库中的配置信息工作。

ovs-dpctl：一个工具，用来配置交换机内核模块，可以控制转发规则。

ovs-vsctl：主要是获取或者更改ovs-vswitchd的配置信息，此工具操作的时候会更新ovsdb-server中的数据库。

ovs-appctl：主要是向OVS守护进程发送命令的，一般用不上。

ovsdbmonitor：GUI工具来显示ovsdb-server中数据信息。

ovs-controller：一个简单的OpenFlow控制器.

ovs-ofctl：用来控制OVS作为OpenFlow交换机工作时候的流表内容。

先看下OVS整体架构，用户空间主要组件有数据库服务ovsdb-server和守护进程ovs-vswitchd。kernel中是datapath内核模块。最上面的Controller表示OpenFlow控制器，控制器与OVS是通过OpenFlow协议进行连接，控制器不一定位于OVS主机上，下面分别介绍图中各组件。

![](/assets/network-virtualnetwork-ovs2.png)

![](/assets/network-virtualnetwork-ovs6.png)

## 2.2 OVS组件详述

### 2.2.1 ovs-vswitchd

ovs-vswitchd守护进程是OVS的核心部件，它和datapath内核模块一起实现OVS基于流的数据交换。作为核心组件，它使用openflow协议与上层OpenFlow控制器通信，使用OVSDB协议与ovsdb-server通信，使用netlink和datapath内核模块通信。ovs-vswitchd在启动时会读取ovsdb-server中配置信息，然后配置内核中的datapaths和所有OVS switches，当ovsdb中的配置信息改变时\(例如使用ovs-vsctl工具\)，ovs-vswitchd也会自动更新其配置以保持与数据库同步。

ovs-vswitchd需要加载datapath内核模块才能正常运行。它会自动配置datapath flows，因此我们不必再使用ovs-dpctl去手动操作datapath，但ovs-dpctl仍可用于调试场合。

在OVS中，ovs-vswitchd从OpenFlow控制器获取流表规则，然后把从datapath中收到的数据包在流表中进行匹配，找到匹配的flows并把所需应用的actions返回给datapath，同时作为处理的一部分，ovs-vswitchd会在datapath中设置一条datapath flows用于后续相同类型的数据包可以直接在内核中执行动作，此datapath flows相当于OpenFlow flows的缓存。对于datapath来说，其并不知道用户空间OpenFlow的存在，datapath内核模块信息如下：

2.2.2 ovsdb-server

ovsdb-server是OVS轻量级的数据库服务，用于整个OVS的配置信息，包括接口/交换内容/VLAN等，OVS主进程ovs-vswitchd根据数据库中的配置信息工作，下面是ovsdb-server进程详细信息。

/etc/openvswitch/conf.db是数据库文件存放位置，文件形式存储保证了服务器重启不会影响其配置信息，ovsdb-server需要文件才能启动，可以使用ovsdb-tool create命令创建并初始化此数据库文件。

--remote=punix:/var/run/openvswitch/db.sock 实现了一个Unix sockets连接，OVS主进程ovs-vswitchd或其它命令工具\(ovsdb-client\)通过此socket连接管理ovsdb。

/var/log/openvswitch/ovsdb-server.log是日志记录。

2.2.3 OpenFlow

OpenFlow是开源的用于管理交换机流表的协议，OpenFlow在OVS中的地位可以参考上面架构图，它是Controller和ovs-vswitched间的通信协议。需要注意的是，OpenFlow是一个独立的完整的流表协议，不依赖于OVS，OVS只是支持OpenFlow协议，有了支持，我们可以使用OpenFlow控制器来管理OVS中的流表，OpenFlow不仅仅支持虚拟交换机，某些硬件交换机也支持OpenFlow协议。

OVS常用作SDN交换机\(OpenFlow交换机\)，其中控制数据转发策略的就是OpenFlow flow。OpenStack Neutron中实现了一个OpenFlow控制器用于向OVS下发OpenFlow flows控制虚拟机间的访问或隔离。本文讨论的默认是作为SDN交换机场景下。

OpenFlow flow的流表项存放于用户空间主进程ovs-vswitchd中，OVS除了连接OpenFlow控制器获取这种flow，文章后面会提到的命令行工具ovs-ofctl工具也可以手动管理OVS中的OpenFlow flow，可以查看man ovs-ofctl了解。

在OVS中，OpenFlow flow是最重要的一种flow, 然而还有其它几种flows存在，文章下面OVS概念部分会提到。

2.2.4 Controller

Controller指OpenFlow控制器。OpenFlow控制器可以通过OpenFlow协议连接到任何支持OpenFlow的交换机，比如OVS。控制器通过向交换机下发流表规则来控制数据流向。除了可以通过OpenFlow控制器配置OVS中flows，也可以使用OVS提供的ovs-ofctl命令通过OpenFlow协议去连接OVS，从而配置flows，命令也能够对OVS的运行状况进行动态监控。

2.2.5 Datapath

在 OpenFlow Switch 规则的语义中，给交换机或者桥，用了一个专业的名词，叫做 Datapath。Open vSwitch 的内核模块 openvswitch.ko 实现了多个 Datapath，每个 Datapath 可以具有多个 Ports。每个 Datapath 通过关联流表（Flow Table）来定义网络包的流向。Datapath 监听网卡接口设备，将监听到的数据包首先在流表中进行匹配，找到匹配的流表项之后把对应的 Actions 返回给 Datapath，作为数据处理行为的描述。Datapath 支持数据在内核空间进行交换。

关于datapath，The Design and Implementation of Open vSwitch中有描述：

The datapath module in the kernel receives the packets first, from a physical NIC or a VM’s virtual NIC. Either ovs-vswitchd has instructed the datapath how to handle packets of this type, or it has not. In the former case, the datapath module simply follows the instructions, called actions, given by ovs-vswitchd, which list physical ports or tunnels on which to transmit the packet. Actions may also specify packet modifications, packet sampling, or instructions to drop the packet. In the other case, where the datapath has not been told what to do with the packet, it delivers it to ovs-vswitchd. In userspace, ovs-vswitchd determines how the packet should be handled, then it passes the packet back to the datapath with the desired handling. Usually, ovs-vswitchd also tells the datapath to cache the actions, for handling similar future packets.

为了说明datapath，来看一张更详细的架构图，图中的大部分组件上面都有提到：

![](/assets/network-virtualnetwork-ovs3.png)

用户空间ovs-vswitchd和内核模块datapath决定了数据包的转发，首先，datapath内核模块收到进入数据包\(物理网卡或虚拟网卡\)，然后查找其缓存\(datapath flows\)，当有一个匹配的flow时它执行对应的操作，否则datapath会把该数据包送入用户空间由ovs-vswitchd负责在其OpenFlow flows中查询\(图1中的First Packet\)，ovs-vswitchd查询后把匹配的actions返回给datapath并设置一条datapath flows到datapath中，这样后续进入的同类型的数据包\(图1中的Subsequent Packets\)因为缓存匹配会被datapath直接处理，不用再次进入用户空间。

datapath专注于数据交换，它不需要知道OpenFlow的存在。与OpenFlow打交道的是ovs-vswitchd，ovs-vswitchd存储所有Flow规则供datapath查询或缓存。

虽然有ovs-dpctl管理工具的存在，但我们没必要去手动管理datapath，这是用户空间ovs-vswitchd的工作。

2.3 Open vSwitch 的工作原理

Bridge 处理数据帧遵循以下几条规则：

在一个 Port 上接收到的帧不会再往此 Port 发送此帧。

接收到的帧都要学习其 Source MAC 地址。

如果数据帧是多播或者广播包（通过 2 层 MAC 地址确定）则要向接收 Port 以外的所有 Port 转发，如果上层协议感兴趣，则还会递交上层处理。

如果数据帧的地址不能在 CAM（MAC-Port Mapping）表中找到，则向接收 Port 以外的所有 Port 转发。

如果 CAM 表中能找到，则转发给相应 Port，如果发送和接收都是同一个 Port，则不发送。

网桥是以混杂模式工作的，所有 MAC 地址的数据帧都能够通过。

用户空间 ovs-vswitchd 和内核模块 Datapath 决定了数据包的转发，如2.2.5节图示：

内核态的 Datapath 监听接口设备流入的数据包。

如果 Datapath 在内核态流表缓存没有找到相应的匹配流表项则将数据包传入（upcall）到用户态的 ovs-vswitchd 守护进程处理。

（可选）用户态的 ovs-vswitchd 拥有完整的流表项，通过 OpenFlow 协议与 OpenFlow 控制器或者 ovs-ofctl 命令行工具进行通信，主要是接收 OpenFlow 控制器南向接口的流表项下发。或者根据流表项设置，ovs-vswitchd 可能会将网络包以 Packet-In 消息发送给 OpenFlow 控制器处理。

ovs-vswitchd 接收到来自 OpenFlow 控制器或 ovs-ofctl 命令行工具的消息后会对内核态的 Flow Table 进行更新。或者根据局部性原理，用户态的 ovs-vswitchd 会将刚刚执行过的 Datapath 没有缓存的流表项注入到 Flow Table 中。

ovs-vswitchd 匹配完流表项之后将数据包重新注入（reinject）到 Datapath。

Datapath 再次访问 Flow Table 获取流表项进行匹配。

最后，网络包被 Datapath 根据流表项 Actions 转发或丢弃。

上述，Datapath 和 ovs-vswitchd 相互配合中包含了两种网络包的处理方式：

Fast Path：Datapatch 加载到内核后，会在网卡上注册一个钩子函数，每当有网络包到达网卡时，这个函数就会被调用，将网络包开始层层拆包（MAC 层，IP 层，TCP 层等），然后与流表项匹配，如果找到匹配的流表项则根据既定策略来处理网络包（e.g. 修改 MAC，修改 IP，修改 TCP 端口，从哪个网卡发出去等等），再将网络包从网卡发出。这个处理过程全在内核完成，所以非常快，称之为 Fast Path。

Slow Path：内核态并没有被分配太多内存，所以内核态能够保存的流表项很少，往往有新的流表项到来后，老的流表项就被丢弃。如果在内核态找不到流表项，则需要到用户态去查询，网络包会通过 netlink（一种内核态与用户态交互的机制）发送给 ovs-vswitchd，ovs-vswitchd 有一个监听线程，当发现有从内核态发过来的网络包，就进入自己的处理流程，然后再次将网络包重新注入到 Datapath。显然，在用户态处理是相对较慢的，故称值为 Slow Path。在用户态的 ovs-vswtichd 不需要吝啬内存，它包含了所有流表项，这些流表项可能是 OpenFlow 控制器通过 OpenFlow 协议下发的，也可能是 OvS 命令行工具 ovs-ofctl 设定的。ovs-vswtichd 会根据网络包的信息层层匹配，直到找到一款流表项进行处理。如果实在找不到，则一般会采用默认流表项，比如丢弃这个包。

当最终匹配到了一个流表项之后，则会根据 “局部性原理（局部数据在一段时间都会被频繁访问，是缓存设计的基础原理）” 再通过 netlink 协议，将这条策略下发到内核态，当这条策略下发给内核时，如果内核的内存空间不足，则会开始淘汰部分老策略。这样保证下一个相同类型的网络包能够直接从内核匹配到，以此加快执行效率。由于近因效应，接下来的网络包应该大概率能够匹配这条策略的。例如：传输一个文件，同类型的网络包会源源不断的到来。

![](/assets/network-virtualnetwork-ovs4.png)



# OVS概念 {#ovs概念}

这部分说下OVS中的重要概念，使用OpenStack neutron+vxlan部署模式下网络节点OVS网桥作为例子

```
# ovs-vsctl show
e44abab7-2f65-4efd-ab52-36e92d9f0200
    Manager "ptcp:6640:127.0.0.1"
        is_connected: true
    Bridge br-ext
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        Port br-ext
            Interface br-ext
                type: internal
        Port "eth1"
            Interface "eth1"
        Port phy-br-ext
            Interface phy-br-ext
                type: patch
                options: {peer=int-br-ext}
    Bridge br-tun
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        Port br-tun
            Interface br-tun
                type: internal
        Port patch-int
            Interface patch-int
                type: patch
                options: {peer=patch-tun}
        Port "vxlan-080058ca"
            Interface "vxlan-080058ca"
                type: vxlan
                options: {df_default="true", in_key=flow, local_ip="8.0.88.201", out_key=flow, remote_ip="8.0.88.202"}
    Bridge br-int
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        Port "qr-11591618-c4"
            tag: 3
            Interface "qr-11591618-c4"
                type: internal
        Port patch-tun
            Interface patch-tun
                type: patch
                options: {peer=patch-int}
        Port int-br-ext
            Interface int-br-ext
                type: patch
                options: {peer=phy-br-ext}
```

### Bridge {#bridge}

Bridge代表一个以太网交换机\(Switch\)，一个主机中可以创建一个或者多个Bridge。Bridge的功能是根据一定规则，把从端口收到的数据包转发到另一个或多个端口，上面例子中有三个Bridge，`br-tun`，`br-int`，`br-ext`

添加一个网桥`br0`

```
ovs-vsctl add-br br0  
```

### Port {#port}

端口Port与物理交换机的端口概念类似，Port是OVS Bridge上创建的一个虚拟端口，每个Port都隶属于一个Bridge。Port有以下几种类型

* **Normal**

可以把操作系统中已有的网卡\(物理网卡em1/eth0,或虚拟机的虚拟网卡tapxxx\)挂载到ovs上，ovs会生成一个同名Port处理这块网卡进出的数据包。此时端口类型为Normal。

如下，主机中有一块物理网卡`eth1`，把其挂载到OVS网桥`br-ext`上，OVS会自动创建同名Port`eth1`。

```
ovs-vsctl add-port br-ext eth1

#Bridge br-ext中出现Port "eth1"
```

有一点要注意的是，挂载到OVS上的网卡设备不支持分配IP地址，因此若之前`eth1`配置有IP地址，挂载到OVS之后IP地址将不可访问。这里的网卡设备不只包括物理网卡，也包括主机上创建的虚拟网卡

* **Internal**

Internal类型是OVS内部创建的虚拟网卡接口，每创建一个Port，OVS会自动创建一个同名接口\(Interface\)挂载到新创建的Port上。接口的概念下面会提到。

下面创建一个网桥br0，并创建一个Internal类型的Port`p0`

```
ovs-vsctl add-br br0   
ovs-vsctl add-port br0 p0 -- set Interface p0 type=internal

#查看网桥br0   
ovs-vsctl show br0
    Bridge "br0"
        fail_mode: secure
        Port "p0"
            Interface "p0"
                type: internal
        Port "br0"
            Interface "br0"
                type: internal
```

可以看到有两个Port。当ovs创建一个新网桥时，默认会创建一个与网桥同名的Internal Port。在OVS中，只有”internal”类型的设备才支持配置IP地址信息，因此我们可以为`br0`接口配置一个IP地址，当然`p0`也可以配置IP地址

```
ip addr add 192.168.10.11/24 dev br0
ip link set br0 up
#添加默认路由
ip route add default via 192.168.10.1 dev br0
```

上面两种Port类型区别在于，Internal类型会自动创建接口\(Interface\)，而Normal类型是把主机中已有的网卡接口添加到OVS中

* **Patch**

当主机中有多个ovs网桥时，可以使用Patch Port把两个网桥连起来。Patch Port总是成对出现，分别连接在两个网桥上，从一个Patch Port收到的数据包会被转发到另一个Patch Port，类似于Linux系统中的`veth`。使用Patch连接的两个网桥跟一个网桥没什么区别，OpenStack Neutron中使用到了Patch Port。上面网桥`br-ext`中的Port`phy-br-ext`与`br-int`中的Port`int-br-ext`是一对Patch Port

可以使用`ovs-vsctl`创建patch设备，如下创建两个网桥`br0,br1`，然后使用一对`Patch Port`连接它们

```
ovs-vsctl add-br br0
ovs-vsctl add-br br1
ovs-vsctl \
-- add-port br0 patch0 -- set interface patch0 type=patch options:peer=patch1 \
-- add-port br1 patch1 -- set interface patch1 type=patch options:peer=patch0

#结果如下
#ovs-vsctl show
    Bridge "br0"
        Port "br0"
            Interface "br0"
                type: internal
        Port "patch0"
            Interface "patch0"
                type: patch
                options: {peer="patch1"}
    Bridge "br1"
        Port "br1"
            Interface "br1"
                type: internal
        Port "patch1"
            Interface "patch1"
                type: patch
                options: {peer="patch0"}
```

连接两个网桥不止上面一种方法，linux中支持创建`Veth`设备对，我们可以首先创建一对`Veth`设备对，然后把这两个`Veth`分别添加到两个网桥上，其效果跟OVS中创建Patch Port一样，只是性能会有差别

```
#创建veth设备对veth-a,veth-b
ip link add veth-a type veth peer name veth-b
#使用Veth连接两个网桥
ovs-vsctl add-port br0 veth-a
ovs-vsctl add-port br1 veth-b
```

* **Tunnel**

OVS中支持添加隧道\(Tunnel\)端口，常见隧道技术有两种`gre`或`vxlan`。隧道技术是在现有的物理网络之上构建一层虚拟网络，上层应用只与虚拟网络相关，以此实现的虚拟网络比物理网络配置更加灵活，并能够实现跨主机的L2通信以及必要的租户隔离。不同隧道技术其大体思路均是将以太网报文使用隧道协议封装，然后使用底层IP网络转发封装后的数据包，其差异性在于选择和构造隧道的协议不同。Tunnel在OpenStack中用作实现大二层网络以及租户隔离，以应对公有云大规模，多租户的复杂网络环境。

OpenStack是多节点结构，同一子网的虚拟机可能被调度到不同计算节点上，因此需要有隧道技术来保证这些同子网不同节点上的虚拟机能够二层互通，就像他们连接在同一个交换机上，同时也要保证能与其它子网隔离。

OVS在计算和网络节点上建立隧道Port来连接各节点上的网桥`br-int`，这样所有网络和计算节点上的`br-int`互联形成了一个大的虚拟的跨所有节点的逻辑网桥\(内部靠tunnel id或VNI隔离不同子网\)，这个逻辑网桥对虚拟机和qrouter是透明的，它们觉得自己连接到了一个大的`br-int`上。从某个计算节点虚拟机发出的数据包会被封装进隧道通过底层网络传输到目的主机然后解封装。

上面网桥`br-tun`中`Port "vxlan-080058ca"`就是一个`vxlan`类型tunnel端口。下面使用两台主机测试创建vxlan隧道

```
#主机192.168.7.21上
ovs-vsctl add-br br-vxlan
#主机192.168.7.23上
ovs-vsctl add-br br-vxlan
#主机192.168.7.21上添加连接到7.23的Tunnel Port
ovs-vsctl add-port br-vxlan tun0 -- set Interface tun0 type=vxlan options:remote_ip=192.168.7.23
#主机192.168.7.23上添加连接到7.21的Tunnel Port
ovs-vsctl add-port br-vxlan tun0 -- set Interface tun0 type=vxlan options:remote_ip=192.168.7.21
```

然后，两个主机上桥接到`br-vxlan`的虚拟机就像连接到同一个交换机一样，可以实现跨主机的L2连接，同时又完全与物理网络隔离。

### Interface {#interface}

Interface是连接到Port的网络接口设备，是OVS与外部交换数据包的组件，在通常情况下，Port和Interface是一对一的关系，只有在配置Port为 bond模式后，Port和Interface是一对多的关系。这个网络接口设备可能是创建`Internal`类型Port时OVS自动生成的虚拟网卡，也可能是系统的物理网卡或虚拟网卡\(TUN/TAP\)挂载在ovs上。 OVS中只有”Internal”类型的网卡接口才支持配置IP地址

`Interface`是一块网络接口设备，负责接收或发送数据包，Port是OVS网桥上建立的一个虚拟端口，`Interface`挂载在Port上。

### Controller {#controller-1}

OpenFlow控制器。OVS可以同时接受一个或者多个OpenFlow控制器的管理。主要作用是下发流表\(Flow Tables\)到OVS，控制OVS数据包转发规则。控制器与OVS通过网络连接，不一定要在同一主机上

可以看到上面实例中三个网桥`br-int`,`br-ext`,`br-tun`都连接到控制器`Controller "tcp:127.0.0.1:6633`上

### datapath {#datapath}

OVS内核模块，负责执行数据交换。其内部有作为缓存使用的flows，上面已经介绍过

# OVS中的各种流\(flows\) {#ovs中的各种流flows}

flows是OVS进行数据转发策略控制的核心数据结构，区别于Linux Bridge是个单纯基于MAC地址学习的二层交换机，flows的存在使OVS作为一款SDN交换机成为云平台网络虚拟机化主要组件

OVS中有多种flows存在，用于不同目的，但最主要的还是OpenFlow flows这种，文中未明确说明的flows都是指OpenFlow flows

### OpenFlow flows {#openflow-flows}

OVS中最重要的一种flows，Controller控制器下发的就是这种flows，OVS架构部分已经简单介绍过，关于openflow的具体使用，会在另一篇文章中说明

### “hidden” flows {#hidden-flows}

OVS在使用OpenFlow flow时，需要与OpenFlow控制器建立TCP连接，若此TCP连接不依赖OVS，即没有OVS依然可以建立连接，此时就是`out-of-band control`模式，这种模式下不需要”hidden” flows

但是在`in-band control`模式下，TCP连接的建立依赖OVS控制的网络，但此时OVS依赖OpenFLow控制器下发的flows才能正常工作，没法建立TCP连接也就无法下发flows，这就产生矛盾了，因此需要存在一些”hidden” flows，这些”hidden” flows保证了TCP连接能够正常建立。关于`in-band control`详细介绍，参考OVS官方文档[Design Decisions In Open vSwitch](https://github.com/openvswitch/ovs/blob/master/Documentation/topics/design.rst)中**In-Band Control**部分

“hidden” flows优先级高于OpenFlow flows，它们不需要手动设置。可以使用`ovs-appctl`查看这些flows，下面命令输出内容包括`OpenFlow flows`,`"hidden" flows`

```
ovs-appctl bridge/dump-flows <br>
```

### datapath flows {#datapath-flows}

datapath flows是`datapath`内核模块维护的flows，由内核模块维护意味着我们并不需要去修改管理它。与OpenFlow flows不同的是，它不支持优先级，并且只有一个表，这些特点使它非常适合做缓存。与OpenFlow一样的是它支持通配符，也支持指令集\(多个action\)

datapath flows可以来自用户空间`ovs-vswitchd`缓存，也可以是datapath内核模块进行MAC地址学习到的flows，这取决与OVS是作为SDN交换机，还是像Linux Bridge那样只是一个简单基于MAC地址学习的二层交换机



2.4 ovs-\*工具的使用及区别

2.4.1 ovs-vsctl

ovs-vsctl是一个管理或配置ovs-vswitchd的高级命令行工具，高级是说其操作对用户友好，封装了对数据库的操作细节。它是管理OVS最常用的命令，除了配置flows之外，其它大部分操作比如Bridge/Port/Interface/Controller/Database/Vlan等都可以完成。![](/assets/network-virtualnetwork-ovs5.png)

2.4.2 ovsdb-tool

ovsdb-tool是一个专门管理OVS数据库文件的工具，不常用，它不直接与ovsdb-server进程通信。![](/assets/network-virtualnetwork-ovsdbtool.png)

2.4.3 ovsdb-client

ovsdb-client是ovsdb-server进程的命令行工具，主要是从正在运行的ovsdb-server中查询信息，操作的是数据库相关。

![](/assets/network-virtualnetwork-ovsdblicnet.png)

2.4.4 ovs-ofctl

ovs-ofctl是专门管理配置OpenFlow交换机的命令行工具，我们可以用它手动配置OVS中的OpenFlow flows，注意其不能操作datapath flows和”hidden” flows。

ovs-vsctl是一个综合的配置管理工具，ovsdb-client倾向于从数据库中查询某些信息，而ovsdb-tool是维护数据库文件工具。

![](/assets/network-virtualnetwork-ovsofctl.png)

