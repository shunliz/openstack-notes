我在第一次听到“层次化端口绑定”时，并没有联想到它对应的真正功能，它是翻译自英文“hierarchical port binding”。这是OpenStack Neutron在很早期就有的功能，但是在OpenStack Neutron里面几乎找不到它的相关文档，甚至代码也只有寥寥几十行。那它究竟是什么？一句话回答，它是为了在VxLAN offload到硬件交换机之后，实现[虚拟机](https://so.csdn.net/so/search?q=%E8%99%9A%E6%8B%9F%E6%9C%BA&spm=1001.2101.3001.7020)多租户隔离的一个功能。

VxLAN有其独特的优势，我在之前的文章\[1\]详细分析过。但是VxLAN也有其不可回避的问题，第一就是VxLAN的封装解封装，消耗较多的CPU，带来的结果是包处理速度下降（进而影响PPS，Latency）；第二就是封装成VxLAN之后，额外的50字节的Overhead对网络带宽的影响。这些都是性能的问题，如果你不关心性能，可以忽略这些问题。但是任何虚拟化技术，性能都会是被challenge的点。对于VxLAN来说，解决办法之一就是将VxLAN 放到（或者说offload到）硬件设备去处理。这样，至少VxLAN的封装解封装不占用服务器CPU，不会影响包处理速度。而虚拟网络能够在享受VxLAN优点的同时，带来接近于无损（相比与非虚拟网络）的网络体验。能够处理VxLAN的硬件设备包括网卡和交换机，层次化端口绑定就是OpenStack为了适配硬件交换机处理VxLAN的场景。

## **交换机中的VTEP**

VxLAN的处理主要包括VxLAN数据的封装和解封装，这些都是由VTEP（VxLAN Tunnel EndPoint）完成的。如果硬件交换机来处理VxLAN，那么就相当于VTEP要在交换机上，VxLAN报文的封装解封装都在交换机上完成。这样的交换机一般是直接与服务器相连接的交换机（ToR，Top of Rack 交换机）。因为VxLAN现在放到了硬件交换机处理，对于服务器或者位于服务器上的虚拟机来说，是感知不到VxLAN的存在。大多数的硬件[SDN](https://so.csdn.net/so/search?q=SDN&spm=1001.2101.3001.7020)（或者说underlay SDN）都是采用这种方式处理VxLAN。

![](https://img-blog.csdnimg.cn/img_convert/f08672e296f7f52baa11da43f59c4fe9.png)

VxLAN的一个最大好处在于，能够提供更多的租户隔离。[多租户](https://so.csdn.net/so/search?q=%E5%A4%9A%E7%A7%9F%E6%88%B7&spm=1001.2101.3001.7020)隔离依靠的是VxLAN Header里面的24bit VNI（Virtual Networking Identity），不同的租户有不同的VNI。但是现在VxLAN的封装解封装都是在硬件交换机完成，租户拥有的服务器或者虚拟机又感知不到VxLAN，硬件交换机怎么知道哪些网络数据属于哪个租户，进而给网络数据分配VNI，完成VxLAN封装呢？

第一种方式是根据交换机端口来区分。例如，交换机端口A的网络流量，认为是租户A的流量，封装成VNI为A的VxLAN数据。同理，端口B的流量认为是租户B的流量，封装成VNI B的VxLAN数据。如下图所示：

![](https://img-blog.csdnimg.cn/img_convert/0d4938b66605f8b4cb15d06366c81a1f.png)

这种方式要求交换机的一个端口只能连接一个租户，也就是说，交换机端口连接的服务器上只能部署一个租户的虚拟机。无论对于管理员还是用户来说，这都不是一个友好的限制。

反过来说，一个服务器如果同时有多个租户的虚拟机，那么交换机的一个端口就会同时存在多个租户的网络数据，也就没有办法再通过交换机端口来区分租户，分配VNI，进而封装VxLAN数据。服务器部署多个租户虚拟机，如下图所示：

![](https://img-blog.csdnimg.cn/img_convert/755d89d65262f6cc192fcf435c4cbe0d.png)

这种情况下该怎么让位于交换机上的VTEP识别多租户，进而封装VxLAN呢？这就需要另外一种标记多租户网络数据的方式。传统[买二手域名](https://www.fgba.net/)网络里面，是通过VLAN识别多租户的网络，这里还可以通过VLAN来做同样的事。首先在服务器内部对不同租户的虚拟机打上不同的VLAN Tag，同时将服务器连接到交换机的Trunk口，这样服务器可以把多个VLAN Tag的网络数据送到交换机。交换机根据VLAN Tag识别不同的租户，进而封装成相应的VxLAN数据。硬件交换机内的VTEP构成如下所示：

![](https://img-blog.csdnimg.cn/img_convert/9591b2e3d7ef09df3e0fcf96de44156c.png)

硬件交换机上的VTEP从Downlink，也就是与服务器连接的Trunk口，收到带 VLAN Tag的网络数据，之后查找“VLAN To VxLAN ID MAP”，找到对应的VxLAN ID，进而封装成VxLAN数据。VTEP L2 Table是VxLAN控制层要填的表，与本文要描述的内容没有关联。

这样，硬件交换机能够完成多个租户的VxLAN封装，而租户虚拟机也不需要知道VxLAN的存在。但是同时，问题也来了。首先，服务器该给不同的租户打什么样的VLAN Tag？其次，交换机里面的VLAN To VxLAN ID MAP由谁来填写？对于OpenStack，是通过层次化端口绑定这个功能来解决这两个问题。

## **层次化端口绑定**

既然在OpenStack内实现这么一个功能，那就需要符合OpenStack的软件架构。层次化端口绑定是在OpenStack Neutron ML2模块中实现的。Neutron ML2我曾在\[2\]中有过介绍。ML2由多类Driver组成，其中一类是Mechanism Driver。每一个Mechanism Driver都管理一种二层网络设备。在层次化端口绑定的场景下，需要两个Mechanism Driver，其中一个管理硬件交换机，另一个管理OpenVSwitch（当然也可以是其他的虚拟交换机，我曾经在VMware DVS上也实现过相同的功能），具体连接图如下所示：

![](https://img-blog.csdnimg.cn/img_convert/2caacc199a7efe0207d3b26624f8e661.png)

原理讲清楚了，具体的连接关系也讲清楚了，接下来的流程就顺利成章了。我们最后来过一下层次化端口绑定的流程。

* 用户创建了一个虚拟机，并且将虚拟机创建在VxLAN A网络中。
* Neutron需要创建一个VxLAN A的网络接口，请求被发送到了ML2。
* Neutron ML2先调用到物理交换机对应的Mechanism driver进行端口绑定（port binding），将VxLAN A与网络接口进行绑定。
* 因为底层还有VLAN，物理交换机的Mechanism Driver会再申请一个VLAN B，并告知Neutron ML2，当前网络接口还需要绑定到对应的VLAN上。
* 之后物理交换机的Mechanism Driver会通过相应的API，告知物理交换机 VLAN B和VxLAN A的对应关系，这样物理交换机就有了VLAN To VxLAN ID MAP。
* ML2因为知道了网络接口还需要绑定到对应的VLAN上，再调用OpenVSwitch的Mechanism Driver，将VLAN B与网络接口进行绑定。
* 之后，OVS的Mechanism Driver会通过相应的API，告知位于计算节点的OpenVSwitch，对于这个网络接口的网络数据，打上VLAN B的Tag。

到此为止层次化端口绑定完成了。在这里，对于同一个网络接口，实际上绑定了两次，一次是在虚拟交换机上的VLAN 绑定，另一次是在硬件交换机上的VxLAN绑定。所以，对于“Hierarchical Port Binding”到层次化端口绑定这个翻译，我个人觉得还是比较符合“信雅达”的标准的。

绑定完成之后，网络数据送到OpenVSwitch，OVS会打上VLAN B的Tag，带VLAN B Tag的网络数据送到物理交换机。物理交换机根据自己的VLAN To VxLAN ID MAP，将VLAN Tag去掉，再封装成相应的VxLAN数据。经过这样的处理，VxLAN的封装解封装被offload到了物理交换机。

那为什么OpenStack Neutron里面没有相应的全部代码？因为层次化端口绑定的逻辑，有一半是在Neutron ML2里面，有另一半是在物理交换机对应的Mechanism driver里面。物理交换机属于各个厂商，相应的Mechanism Driver由各个厂商维护，而OpenStack Neutron不包含各个厂商的代码。所以，有关层次化端口绑定的代码，在OpenStack Neutron中是看不到完整的



\[1\] https://zhuanlan.zhihu.com/p/36165475

\[2\] https://zhuanlan.zhihu.com/p/29833977

\[3\] https://github.com/openstack/neutron/blob/dd14501c12243a8e5fb4086ba461482e54ac75bc/neutron/plugins/ml2/managers.py\#L760-L824

\[4\] https://github.com/openstack/networking-cisco/blob/aa58a30aec25b86f9aa5952b0863045975debfa9/networking\_cisco/ml2\_drivers/nexus/mech\_cisco\_nexus.py\#L1842-L1905

