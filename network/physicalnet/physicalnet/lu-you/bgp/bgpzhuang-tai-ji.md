# BGP\(边界网关协议\)-----状态机

BGP 对等体的建立、更新和删除等交互过程主要有 5 种报文、6 种状态机和 5 个原则。![](/assets/network-basic-routing-bgpstate1.png)![](/assets/network-basic-routing-bgpstate1_1.png)

![](/assets/network-basic-routing-bgpstate2.png)

![](/assets/network-basic-routing-bgpstate3.png)![](/assets/network-basic-routing-bgpstate4.png)

**BGP交互原则补充：**

```
1、BGP设备将最优路由加入BGP路由表，形成BGP路由。
   BGP设备与对等体建立邻居关系后，采取以下交互原则

2、从IBGP对等体获得的BGP路由，BGP设备只发布给它的EBGP对等体

3、从EBGP对等体获得的BGP路由，BGP设备发布给它所有EBGP和IBGP对等体

4、当存在多条到达同一目的地址的有效路由时，BGP设备只将最优路由发布给对等体

5、路由更新时，BGP设备只发送更新的BGP路由

6、所有对等体发送的路由，BGP设备都会接收

7、所有EBGP对等体在传递过程中下一跳改变

8、所有IBGP对等体在传递过程中下一跳不变

9、默认EBGP传递时TTL值为1

10、默认IBGP传递时TTL值为255

11、建立IBGP对等体时要让下一跳可达，处于边界的IBGP对等体需要将下一跳指向自己，
    这样才能建立IBGP对等体

12、用环回口建立邻居需要注意的点
需要修改更新源，默认更新源是物理口，需要修改成环回口。
建立IBGP对等体时要保障下一跳可达，处于边界的IBGP对等体需要将下一跳指向自己，
这样才能建立IBGP对等体

建立EBGP对等体时因为EBGP只能传一跳，因而，在建立EBGP对等体时，
需要修改EBGP多跳的跳数为2以上（自己环回到对端环回是两跳，默认一跳）

13、关于为什么要用环回口建邻居

原因是环回口稳定，只要路由器启动着，环回口就不DOWN，
而物理链路可能会受线路或者接口等因素的影响导致对等体关系有问题，
因而一般BGP建立对等体都是环回口来建
```

**BGP与IGP交互：**

BGP与IGP在设备中使用不同的路由表，为了实现不同AS间相互通讯，BGP需要与IGP进行交互，即BGP路由表和IGP路由表相互引入。



**BGP引入IGP路由：**



BGP协议本身不发现路由，因此需要将其他路由引入到BGP路由表，实现AS间的路由互通。当一个AS需要将路由发布给其他AS时，AS边缘路由器会在BGP路由表中引入IGP的路由。为了更好的规划网络，BGP在引入IGP的路由时，可以使用路由策略进行路由过滤和路由属性设置，也可以设置MED值指导EBGP对等体判断流量进入AS时选路。

BGP引入路由时支持Import和Network两种方式：

1、Import方式是按协议类型，将RIP、OSPF、ISIS等协议的路由引入到BGP路由表中。为了保证引入的IGP路由的有效性，Import方式还可以引入静态路由和直连路由。

2、Network方式是逐条将IP路由表中已经存在的路由引入到BGP路由表中，比Import方式更精确。



**IGP引入BGP路由：**



当一个AS需要引入其他AS的路由时，AS边缘路由器会在IGP路由表中引入BGP的路由。为了避免大量BGP路由对AS内设备造成影响，当IGP引入BGP路由时，可以使用路由策略，进行路由过滤和路由属性设置。



**bgp与ospf,isis关系**

ospf用来计算路由，选一条最优路径

bgp只负责路由传递

![](/assets/network-basic-routing-bgpstate5.png)





bgp优势：



1、bgp因为基于TCP，既可靠又快

2、bgp是距离矢量，只会传路由，压根不会传拓扑信息。距离矢量不怎么占资源，有什么路由传什么路由，不会像链路状态一样，先计算拓扑信息再传路由。

3、bgp只能触发式更新，不能周期性更新

（只有第一次建立连接时的update报文是全量，后续都是增量）

