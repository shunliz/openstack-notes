OSPF（Open Shortest Path First，开放最短路径优先）是IETF（Internet Engineering Task Force，互联网工程任务组）组织开发的一个基于链路状态的内部网关协议。目前针对IPv4协议使用的是OSPF Version 2。

## [OSPF]()的特点

OSPF具有如下特点：

* 适应范围广：支持各种规模的网络，最多可支持几百台路由器。
* 快速收敛：在网络的拓扑结构发生变化后立即发送更新报文，使这一变化在自治系统中同步。
* 无自环：由于OSPF根据收集到的链路状态用最短路径树算法计算路由，从算法本身保证了不会生成自环路由。
* 区域划分：允许自治系统的网络被划分成区域来管理。路由器链路状态数据库的减小降低了内存的消耗和CPU的负担；区域间传送路由信息的减少降低了网络带宽的占用。
* 等价路由：支持到同一目的地址的多条等价路由。
* 路由分级：使用4类不同的路由，按优先顺序来说分别是：区域内路由、区域间路由、第一类外部路由、第二类外部路由。
* 支持验证：支持基于接口的报文验证，以保证报文交互和路由计算的安全性。
* 组播发送：在某些类型的链路上以组播地址发送协议报文，减少对其他设备的干扰。

## [OSPF]()的基本概念

#### 1.自治系统（Autonomous System）

一组使用相同路由协议交换路由信息的路由器，缩写为AS。

#### 2.OSPF路由的计算过程

OSPF协议路由的计算过程可简单描述如下：

* 每台OSPF路由器根据自己周围的网络拓扑结构生成LSA（Link State Advertisement，链路状态通告），并通过更新报文将LSA发送给网络中的其它OSPF路由器。
* 每台OSPF路由器都会收集其它路由器通告的LSA，所有的LSA放在一起便组成了LSDB（Link State Database，链路状态数据库）。LSA是对路由器周围网络拓扑结构的描述，LSDB则是对整个自治系统的网络拓扑结构的描述。
* OSPF路由器将LSDB转换成一张带权的有向图，这张图便是对整个网络拓扑结构的真实反映。各个路由器得到的有向图是完全相同的。
* 每台路由器根据有向图，使用SPF算法计算出一棵以自己为根的最短路径树，这棵树给出了到自治系统中各节点的路由。

#### 3.路由器ID号

一台路由器如果要运行OSPF协议，则必须存在RID（Router ID，路由器ID）。RID是一个32比特无符号整数，可以在一个自治系统中唯一的标识一台路由器。

RID可以手工配置，也可以自动生成；如果没有通过命令指定RID，将按照如下顺序自动生成一个RID：

* 如果当前设备配置了Loopback接口，将选取所有Loopback接口上数值最大的IP地址作为RID；
* 如果当前设备没有配置Loopback接口，将选取它所有已经配置IP地址且链路有效的接口上数值最大的IP地址作为RID。

#### 4.OSPF的协议报文

OSPF有五种类型的协议报文：

* Hello报文：周期性发送，用来发现和维持OSPF邻居关系。内容包括一些定时器的数值、DR（Designated Router，指定路由器）、BDR（Backup Designated Router，备份指定路由器）以及自己已知的邻居。
* DD（Database Description，数据库描述）报文：描述了本地LSDB中每一条LSA的摘要信息，用于两台路由器进行数据库同步。
* LSR（Link State Request，链路状态请求）报文：向对方请求所需的LSA。两台路由器互相交换DD报文之后，得知对端的路由器有哪些LSA是本地的LSDB所缺少的，这时需要发送LSR报文向对方请求所需的LSA。内容包括所需要的LSA的摘要。
* LSU（Link State Update，链路状态更新）报文：向对方发送其所需要的LSA。
* LSAck（Link State Acknowledgment，链路状态确认）报文：用来对收到的LSA进行确认。内容是需要确认的LSA的Header（一个报文可对多个LSA进行确认）。

#### 5.LSA的类型

OSPF中对链路状态信息的描述都是封装在LSA中发布出去，常用的LSA有以下几种类型：

* Router LSA（Type1）：由每个路由器产生，描述路由器的链路状态和开销，在其始发的区域内传播。
* Network LSA（Type2）：由DR产生，描述本网段所有路由器的链路状态，在其始发的区域内传播。
* Network Summary LSA（Type3）：由ABR（Area Border Router，区域边界路由器）产生，描述区域内某个网段的路由，并通告给其他区域。
* ASBR Summary LSA（Type4）：由ABR产生，描述到ASBR（Autonomous System Boundary Router，自治系统边界路由器）的路由，通告给相关区域。
* AS External LSA（Type5）：由ASBR产生，描述到AS（Autonomous System，自治系统）外部的路由，通告到所有的区域（除了Stub区域和NSSA区域）。
* NSSA External LSA（Type7）：由NSSA（Not-So-Stubby Area）区域内的ASBR产生，描述到AS外部的路由，仅在NSSA区域内传播。
* Opaque LSA：是一个被提议的LSA类别，由标准的LSA头部后面跟随特殊应用的信息组成，可以直接由OSPF协议使用，或者由其它应用分发信息到整个OSPF域间接使用。Opaque LSA分为Type 9、Type10、Type11三种类型，泛洪区域不同；其中，Type 9的Opaque LSA仅在本地链路范围进行泛洪，Type 10的Opaque LSA仅在本地区域范围进行泛洪，Type 11的LSA可以在一个自治系统范围进行泛洪。

#### 6.邻居和邻接

在OSPF中，邻居（Neighbor）和邻接（Adjacency）是两个不同的概念。

OSPF路由器启动后，便会通过OSPF接口向外发送Hello报文。收到Hello报文的OSPF路由器会检查报文中所定义的参数，如果双方一致就会形成邻居关系。

形成邻居关系的双方不一定都能形成邻接关系，这要根据网络类型而定。只有当双方成功交换DD报文，交换LSA并达到LSDB的同步之后，才形成真正意义上的邻接关系。

## [OSPF]()区域与路由聚合

#### 1.区域划分

随着网络规模日益扩大，当一个大型网络中的路由器都运行OSPF路由协议时，路由器数量的增多会导致LSDB非常庞大，占用大量的存储空间，并使得运行SPF算法的复杂度增加，导致CPU负担很重。

在网络规模增大之后，拓扑结构发生变化的概率也增大，网络会经常处于“振荡”之中，造成网络中会有大量的OSPF协议报文在传递，降低了网络的带宽利用率。更为严重的是，每一次变化都会导致网络中所有的路由器重新进行路由计算。

OSPF协议通过将自治系统划分成不同的区域（Area）来解决上述问题。区域是从逻辑上将路由器划分为不同的组，每个组用区域号（Area ID）来标识。区域的边界是路由器，而不是链路。一个网段（链路）只能属于一个区域，或者说每个运行OSPF的接口必须指明属于哪一个区域。如图1所示。

![](/assets/network-basic-route-ospf1.png)

划分区域后，可以在区域边界路由器上进行路由聚合，以减少通告到其他区域的LSA数量，还可以将网络拓扑变化带来的影响最小化。

#### 2.路由器的类型

OSPF路由器根据在AS中的不同位置，可以分为以下四类：

\(1\)区域内路由器（Internal Router）

该类路由器的所有接口都属于同一个OSPF区域。

\(2\)区域边界路由器ABR（Area Border Router）

该类路由器可以同时属于两个以上的区域，但其中一个必须是骨干区域（骨干区域的介绍请参见下一小节）。ABR用来连接骨干区域和非骨干区域，它与骨干区域之间既可以是物理连接，也可以是逻辑上的连接。

\(3\)骨干路由器（Backbone Router）

该类路由器至少有一个接口属于骨干区域。因此，所有的ABR和位于Area0的内部路由器都是骨干路由器。

\(4\)自治系统边界路由器ASBR

与其他AS交换路由信息的路由器称为ASBR。ASBR并不一定位于AS的边界，它有可能是区域内路由器，也有可能是ABR。只要一台OSPF路由器引入了外部路由的信息，它就成为ASBR。

![](/assets/network-basic-route-ospf2.png)

#### 3.骨干区域与虚连接

OSPF划分区域之后，并非所有的区域都是平等的关系。其中有一个区域是与众不同的，它的区域号（Area ID）是0，通常被称为骨干区域。骨干区域负责区域之间的路由，非骨干区域之间的路由信息必须通过骨干区域来转发。对此，OSPF有两个规定：

* 所有非骨干区域必须与骨干区域保持连通；
* 骨干区域自身也必须保持连通。

但在实际应用中，可能会因为各方面条件的限制，无法满足这个要求。这时可以通过配置OSPF虚连接（Virtual Link）予以解决。

虚连接是指在两台ABR之间通过一个非骨干区域而建立的一条逻辑上的连接通道。它的两端必须是ABR，而且必须在两端同时配置方可生效。为虚连接两端提供一条非骨干区域内部路由的区域称为传输区（Transit Area）。

在图3中，Area2与骨干区域之间没有直接相连的物理链路，但可以在ABR上配置虚连接，使Area2通过一条逻辑链路与骨干区域保持连通。

![](/assets/network-baisc-route-ospf3.png)

虚连接的另外一个应用是提供冗余的备份链路，当骨干区域因链路故障不能保持连通时，通过虚连接仍然可以保证骨干区域在逻辑上的连通性。如图4所示。

![](/assets/network-basic-route-ospf4.png)

虚连接相当于在两个ABR之间形成了一个点到点的连接，因此，在这个连接上，和物理接口一样可以配置接口的各参数，如发送Hello报文间隔等。

两台ABR之间直接传递OSPF报文信息，它们之间的OSPF路由器只是起到一个转发报文的作用。由于协议报文的目的地址不是中间这些路由器，所以这些报文对于它们而言是透明的，只是当作普通的IP报文来转发。

#### 4.\(Totally\) Stub区域

Stub区域是一些特定的区域，Stub区域的ABR不允许注入Type5 LSA，在这些区域中路由器的路由表规模以及路由信息传递的数量都会大大减少。

为了进一步减少Stub区域中路由器的路由表规模以及路由信息传递的数量，可以将该区域配置为Totally Stub（完全Stub）区域，该区域的ABR不会将区域间的路由信息和外部路由信息传递到本区域。

\(Totally\) Stub区域是一种可选的配置属性，但并不是每个区域都符合配置的条件。通常来说，\(Totally\) Stub区域位于自治系统的边界。

为保证到本自治系统的其他区域或者自治系统外的路由依旧可达，该区域的ABR将生成一条缺省路由，并发布给本区域中的其他非ABR路由器。

配置\(Totally\) Stub区域时需要注意下列几点：

* 骨干区域不能配置成\(Totally\) Stub区域。
* 如果要将一个区域配置成\(Totally\) Stub区域，则该区域中的所有路由器必须都要配置**stub**\[**no-summary**\]命令。
* \(Totally\) Stub区域内不能存在ASBR，即自治系统外部的路由不能在本区域内传播。
* 虚连接不能穿过\(Totally\) Stub区域。

#### 5.NSSA区域

NSSA（Not-So-Stubby Area）区域是Stub区域的变形，与Stub区域有许多相似的地方。NSSA区域也不允许Type5 LSA注入，但可以允许Type7 LSA注入。Type7 LSA由NSSA区域的ASBR产生，在NSSA区域内传播。当Type7 LSA到达NSSA的ABR时，由ABR将Type7 LSA转换成Type5 LSA，传播到其他区域。

如图5所示，运行OSPF协议的自治系统包括3个区域：区域1、区域2和区域0，另外两个自治系统运行RIP协议。区域1被定义为NSSA区域，区域1接收的RIP路由传播到NSSA ASBR后，由NSSA ASBR产生Type7 LSA在区域1内传播，当Type7 LSA到达NSSA ABR后，转换成Type5 LSA传播到区域0和区域2。

另一方面，运行RIP的自治系统的RIP路由通过区域2的ASBR产生Type5 LSA在OSPF自治系统中传播。但由于区域1是NSSA区域，所以Type5 LSA不会到达区域1。

与Stub区域一样，虚连接也不能穿过NSSA区域。

![](/assets/network-basic-route-ospf5.png)

#### 6.路由聚合

路由聚合是指ABR或ASBR将具有相同前缀的路由信息聚合，只发布一条路由到其它区域。

AS被划分成不同的区域后，区域间可以通过路由聚合来减少路由信息，减小路由表的规模，提高路由器的运算速度。

例如，图6中，Area 1内有三条区域内路由19.1.1.0/24，19.1.2.0/24，19.1.3.0/24，如果此时在Router A上配置了路由聚合，将三条路由聚合成一条19.1.0.0/16，则Router A就只生成一条聚合后的LSA，并发布给Area0中的其他路由器。

![](/assets/network-basic-route-ospf6.png)

OSPF有两类聚合：

\(1\)ABR聚合

ABR向其它区域发送路由信息时，以网段为单位生成Type3 LSA。如果该区域中存在一些连续的网段，则可以将这些连续的网段聚合成一个网段。这样ABR只发送一条聚合后的LSA，所有属于聚合网段范围的LSA将不再会被单独发送出去，这样可减少其它区域中LSDB的规模。

[\(2\)ASBR]()聚合

配置引入路由聚合后，如果本地路由器是自治系统边界路由器ASBR，将对引入的聚合地址范围内的Type5 LSA进行聚合。当配置了NSSA区域时，还要对引入的聚合地址范围内的Type7 LSA进行聚合。

如果本地路由器是ABR，则对由Type7 LSA转化成的Type5 LSA进行聚合处理。

#### [7.路由类型]()

OSPF将路由分为四类，按照优先级从高到低的顺序依次为：

* 区域内路由（Intra Area）
* 区域间路由（Inter Area）
* 第一类外部路由（Type1 External）：这类路由的可信程度较高，并且和OSPF自身路由的开销具有可比性，所以到第一类外部路由的开销等于本路由器到相应的ASBR的开销与ASBR到该路由目的地址的开销之和。
* 第二类外部路由（Type2 External）：这类路由的可信度比较低，所以OSPF协议认为从ASBR到自治系统之外的开销远远大于在自治系统之内到达ASBR的开销。所以计算路由开销时将主要考虑前者，即到第二类外部路由的开销等于ASBR到该路由目的地址的开销。如果计算出开销值相等的两条路由，再考虑本路由器到相应的ASBR的开销

区域内和区域间路由描述的是AS内部的网络结构，外部路由则描述了应该如何选择到AS以外目的地址的路由。

## [OSPF]()的网络类型

#### 1.OSPF的4种网络类型

OSPF根据链路层协议类型将网络分为下列四种类型：

* Broadcast：当链路层协议是Ethernet、FDDI时，OSPF缺省认为网络类型是Broadcast。在该类型的网络中，通常以组播形式（224.0.0.5和224.0.0.6）发送协议报文。
* NBMA（Non-Broadcast Multi-Access，非广播多路访问）：当链路层协议是帧中继、ATM或X.25时，OSPF缺省认为网络类型是NBMA。在该类型的网络中，以单播形式发送协议报文。
* P2MP（Point-to-MultiPoint，点到多点）：没有一种链路层协议会被缺省的认为是P2MP类型。点到多点必须是由其他的网络类型强制更改的。常用做法是将NBMA改为点到多点的网络。在该类型的网络中，以组播形式（224.0.0.5）发送协议报文。
* P2P（Point-to-Point，点到点）：当链路层协议是PPP、HDLC时，OSPF缺省认为网络类型是P2P。在该类型的网络中，以组播形式（224.0.0.5）发送协议报文。

#### 2.NBMA网络的配置原则

NBMA网络是指非广播、多点可达的网络，比较典型的有ATM和帧中继网络。

对于接口的网络类型为NBMA的网络需要进行一些特殊的配置。由于无法通过广播Hello报文的形式发现相邻路由器，必须手工为该接口指定相邻路由器的IP地址，以及该相邻路由器是否有DR选举权等。

NBMA网络必须是全连通的，即网络中任意两台路由器之间都必须有一条虚电路直接可达。如果部分路由器之间没有直接可达的链路时，应将接口配置成P2MP类型。如果路由器在NBMA网络中只有一个对端，也可将接口类型配置为P2P类型。

NBMA与P2MP网络之间的区别如下：

* NBMA网络是指那些全连通的、非广播、多点可达网络。而P2MP网络，则并不需要一定是全连通的。
* 在NBMA网络中需要选举DR与BDR，而在P2MP网络中没有DR与BDR。
* NBMA是一种缺省的网络类型，而P2MP网络必须是由其它的网络强制更改的。最常见的做法是将NBMA网络改为P2MP网络。
* NBMA网络采用单播发送报文，需要手工配置邻居。P2MP网络采用组播方式发送报文。

## [DR]()[/BDR]()

#### 1.DR/BDR简介

在广播网和NBMA网络中，任意两台路由器之间都要交换路由信息。如果网络中有n台路由器，则需要建立n\(n-1\)/2个邻接关系。这使得任何一台路由器的路由变化都会导致多次传递，浪费了带宽资源。为解决这一问题，OSPF协议定义了指定路由器DR（Designated Router），所有路由器都只将信息发送给DR，由DR将网络链路状态发送出去。

如果DR由于某种故障而失效，则网络中的路由器必须重新选举DR，再与新的DR同步。这需要较长的时间，在这段时间内，路由的计算是不正确的。为了能够缩短这个过程，OSPF提出了BDR（Backup Designated Router，备份指定路由器）的概念。

BDR实际上是对DR的一个备份，在选举DR的同时也选举出BDR，BDR也和本网段内的所有路由器建立邻接关系并交换路由信息。当DR失效后，BDR会立即成为DR。由于不需要重新选举，并且邻接关系事先已建立，所以这个过程是非常短暂的。当然这时还需要再重新选举出一个新的BDR，虽然一样需要较长的时间，但并不会影响路由的计算。

DR和BDR之外的路由器（称为DR Other）之间将不再建立邻接关系，也不再交换任何路由信息。这样就减少了广播网和NBMA网络上各路由器之间邻接关系的数量。

如图7所示，用实线代表以太网物理连接，虚线代表建立的邻接关系。可以看到，采用DR/BDR机制后，5台路由器之间只需要建立7个邻接关系就可以了。

![](/assets/network-basic-route-ospf7.png)

#### 2.DR/BDR选举过程

DR和BDR是由同一网段中所有的路由器根据路由器优先级、Router ID通过HELLO报文选举出来的，只有优先级大于0的路由器才具有选取资格。

进行DR/BDR选举时每台路由器将自己选出的DR写入Hello报文中，发给网段上的每台运行OSPF协议的路由器。当处于同一网段的两台路由器同时宣布自己是DR时，路由器优先级高者胜出。如果优先级相等，则Router ID大者胜出。如果一台路由器的优先级为0，则它不会被选举为DR或BDR。

需要注意的是：

* 只有在广播或NBMA类型接口才会选举DR，在点到点或点到多点类型的接口上不需要选举DR。
* DR是某个网段中的概念，是针对路由器的接口而言的。某台路由器在一个接口上可能是DR，在另一个接口上有可能是BDR，或者是DR Other。
* 路由器的优先级可以影响一个选取过程，但是当DR/BDR已经选取完毕，就算一台具有更高优先级的路由器变为有效，也不会替换该网段中已经选取的DR/BDR成为新的DR/BDR。
* DR并不一定就是路由器优先级更高的路由器接口；同理，BDR也并不一定就是路由器优先级次高的路由器接口。

## [OSPF]()的报文格式

OSPF报文直接封装为IP报文协议报文，协议号为89。一个比较完整的OSPF报文（以LSU报文为例）结构如图8所示。

![](/assets/network-basic-route-ospf8.png)

#### 1.OSPF报文头

OSPF有五种报文类型，它们有相同的报文头。如图9所示。

![](/assets/network-basic-route-ospf9.png)

主要字段的解释如下：

* Version：OSPF的版本号。对于OSPFv2来说，其值为2。
* Type：OSPF报文的类型。数值从1到5，分别对应Hello报文、DD报文、LSR报文、LSU报文和LSAck报文。
* Packet length：OSPF报文的总长度，包括报文头在内，单位为字节。
* Router ID：始发该LSA的路由器的ID。
* Area ID：始发LSA的路由器所在的区域ID。
* Checksum：对整个报文的校验和。
* AuType：验证类型。可分为不验证、简单（明文）口令验证和MD5验证，其值分别为0、1、2。
* Authentication：其数值根据验证类型而定。当验证类型为0时未作定义，为1时此字段为密码信息，类型为2时此字段包括Key ID、MD5验证数据长度和序列号的信息。

`&说明：`

`MD5验证数据添加在OSPF报文后面，不包含在Authenticaiton字段中。`



#### 2.Hello报文（Hello Packet）

最常用的一种报文，周期性的发送给邻居路由器用来维持邻居关系以及DR/BDR的选举，内容包括一些定时器的数值、DR、BDR以及自己已知的邻居。Hello报文格式如图10所示。

![](/assets/network-basic-route-ospf10.png)

主要字段解释如下：

* Network Mask：发送Hello报文的接口所在网络的掩码，如果相邻两台路由器的网络掩码不同，则不能建立邻居关系。
* HelloInterval：发送Hello报文的时间间隔。如果相邻两台路由器的Hello间隔时间不同，则不能建立邻居关系。
* Rtr Pri：路由器优先级。如果设置为0，则该路由器接口不能成为DR/BDR。
* RouterDeadInterval：失效时间。如果在此时间内未收到邻居发来的Hello报文，则认为邻居失效。如果相邻两台路由器的失效时间不同，则不能建立邻居关系。
* Designated Router：指定路由器的接口的IP地址。
* Backup Designated Router：备份指定路由器的接口的IP地址。
* Neighbor：邻居路由器的Router ID。

#### 3.DD报文（Database Description Packet）

两台路由器进行数据库同步时，用DD报文来描述自己的LSDB，内容包括LSDB中每一条LSA的Header（LSA的Header可以唯一标识一条LSA）。LSA Header只占一条LSA的整个数据量的一小部分，这样可以减少路由器之间的协议报文流量，对端路由器根据LSA Header就可以判断出是否已有这条LSA。

DD报文格式如图11所示。

![](/assets/network-basic-route-ospf11.png)

主要字段的解释如下：

* Interface MTU：在不分片的情况下，此接口最大可发出的IP报文长度。
* I（Initial）：当发送连续多个DD报文时，如果这是第一个DD报文，则置为1，否则置为0。
* M（More）：当连续发送多个DD报文时，如果这是最后一个DD报文，则置为0。否则置为1，表示后面还有其他的DD报文。
* MS（Master/Slave）：当两台OSPF路由器交换DD报文时，首先需要确定双方的主（Master）从（Slave）关系，Router ID大的一方会成为Master。当值为1时表示发送方为Master。
* DD Sequence Number：DD报文序列号，由Master方规定起始序列号，每发送一个DD报文序列号加1，Slave方使用Master的序列号作为确认。主从双方利用序列号来保证DD报文传输的可靠性和完整性。

#### 4.LSR报文（Link State Request Packet）

两台路由器互相交换过DD报文之后，知道对端的路由器有哪些LSA是本地的LSDB所缺少的，这时需要发送LSR报文向对方请求所需的LSA。内容包括所需要的LSA的摘要。LSR报文格式如图12所示。

![](/assets/network-basic-route-ospf12.png)

主要字段解释如下：

* LS type：LSA的类型号。例如Type1表示Router LSA。
* Link State ID：链路状态标识，根据LSA的类型而定。
* Advertising Router：产生此LSA的路由器的Router ID。

#### 5.LSU报文（Link State Update Packet）

LSU报文用来向对端路由器发送所需要的LSA，内容是多条LSA（全部内容）的集合。LSU报文格式如图13所示。

![](/assets/network-basic-route-ospf13.png)

主要字段解释如下：

* Number of LSAs：该报文包含的LSA的数量。
* LSAs：该报文包含的所有LSA。

#### 6.LSAck报文（Link State Acknowledgment Packet）

LSAck报文用来对接收到的LSU报文进行确认，内容是需要确认的LSA的Header。一个LSAck报文可对多个LSA进行确认。报文格式如图14所示。

![](/assets/network-basic-route-ospf14.png)

主要字段解释如下：

* LSA Headers：该报文包含的LSA头部。

#### 7.LSA头格式

所有的LSA都有相同的报文头，其格式如图15所示。

![](/assets/network-basic-route-ospf15.png)

主要字段的解释如下：

* LS age：LSA产生后所经过的时间，以秒为单位。LSA在本路由器的链路状态数据库（LSDB）中会随时间老化（每秒钟加1），但在网络的传输过程中却不会。
* LS type：LSA的类型。
* Link State ID：具体数值根据LSA的类型而定。
* Advertising Router：始发LSA的路由器的ID。
* LS sequence number：LSA的序列号，其他路由器根据这个值可以判断哪个LSA是最新的。
* LS checksum：除了LS age字段外，关于LSA的全部信息的校验和。
* length：LSA的总长度，包括LSA Header，以字节为单位。

## [LSA]()类型

\(1\)Router LSA

![](/assets/network-basic-route-ospf16.png)

主要字段的解释如下：

* Link State ID：产生此LSA的路由器的Router ID。
* V（Virtual Link）：如果产生此LSA的路由器是虚连接的端点，则置为1。
* E（External）：如果产生此LSA的路由器是ASBR，则置为1。
* B（Border）：如果产生此LSA的路由器是ABR，则置为1。
* \# links：LSA中所描述的链路信息的数量，包括路由器上处于某区域中的所有链路和接口。
* Link ID：链路标识，具体的数值根据链路类型而定。
* Link Data：链路数据，具体的数值根据链路类型而定。
* Type：链路类型，取值为1表示通过点对点链路与另一路由器相连，取值为2表示连接到传送网络，取值为3表示连接到Stub网络，取值为4表示虚连接。
* \#TOS：描述链路的不同方式的数量。
* metric：链路的开销。
* TOS：服务类型。
* TOS metric：指定服务类型的链路的开销。

\(2\)Network LSA

Network LSA由广播网或NBMA网络中的DR发出，LSA中记录了这一网段上所有路由器的Router ID。

![](/assets/network-basic-route-ospf17.png)

主要字段的解释如下：

* Link State ID：DR的IP地址。
* Network Mask：广播网或NBMA网络地址的掩码。
* Attached Router：连接在同一个网段上的所有与DR形成了完全邻接关系的路由器的Router ID，也包括DR自身的Router ID。

\(3\)Summary LSA

Network Summary LSA（Type3 LSA）和ASBR Summary LSA（Type4 LSA）除Link State ID字段有所不同外，有着相同的格式，它们都是由ABR产生。

![](/assets/network-basic-route-ospf18.png)

主要字段的解释如下：

* Link State ID：对于Type3 LSA来说，它是所通告的区域外的网络地址；对于Type4来说，它是所通告区域外的ASBR的Router ID。
* Network Mask：Type3 LSA的网络地址掩码。对于Type4 LSA来说没有意义，设置为0.0.0.0。
* metric：到目的地址的路由开销。

`&说明：`

`Type3的LSA可以用来通告缺省路由，此时Link State ID和Network Mask都设置为0.0.0.0。`

\(4\)AS External LSA

由ASBR产生，描述到AS外部的路由信息。

![](/assets/network-basic-route-ospf19.png)

主要字段的解释如下：

* Link State ID：所要通告的其他外部AS的目的地址，如果通告的是一条缺省路由，那么链路状态ID（Link State ID）和网络掩码（Network Mask）字段都将设置为0.0.0.0。
* Network Mask：所通告的目的地址的掩码。
* E（External Metric）：外部度量值的类型。如果是第2类外部路由就设置为1，如果是第1类外部路由则设置为0。关于外部路由类型的详细描述请参见7.路由类型部分。
* metirc：路由开销。
* Forwarding Address：到所通告的目的地址的报文将被转发到的地址。
* External Route Tag：添加到外部路由上的标记。OSPF本身并不使用这个字段，它可以用来对外部路由进行管理。

\(5\)NSSA External LSA

由NSSA区域内的ASBR产生，且只能在NSSA区域内传播。其格式与AS External LSA相同，如图20所示。

![](/assets/network-basic-route-ospf20.png)

