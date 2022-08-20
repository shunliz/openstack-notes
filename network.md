# 传统网络到虚拟化网络的演进

# 传统网络![](/assets/network-traditional-network.png)虚拟网络![](/assets/network-virtualnetwork.png)分布式虚拟网络![](/assets/network-distributedvirtualnetwork.png)

# 单一平面网络到混合平面网络的演进

**单一平面租户共享网络**：所有租户共享一个网络（IP 地址池），只能存在单一网络类型（VLAN 或 Flat）。

* 没有私有网络
* 没有租户隔离
* 没有三层路由

![](/assets/network-single-planenetwork.png)

**多平面租户共享网络**：具有多个共享网络供租户选择。

* 没有私有网络
* 没有租户隔离
* 没有三层路由

![](/assets/network-multipletenantnetwork.png)

**混合平面（共享/私有）网络**：共享网络和租户私有网络混合。

* 有私有网络
* 没有租户隔离
* 没有三层路由

NOTE：因为多租户之间还是依赖共享网络（e.g. 需要访问外部网络），没有做到完全的租户隔离。

![](/assets/network-mixedmultiplenetwork.png)

**基于运营商路由功能的多平面租户私有网络**：每个租户拥有自己私有的网络，不同的网络之间通过运营商路由器（公共）来实现三层互通。

* 有私有网络
* 有租户隔离
* 共享三层路由

![](/assets/network-sproutingmultitntprvnetwork.png)

**基于私有路由器实现的多平面租户私有多网络**：每个租户可以拥有若干个私有网络及路由器。租户可以利用私有路由器实现私有网络间的三层互通。

* 有私有网络
* 有租户隔离
* 有三层路由

![](/assets/network-privateroutingmultitntprivnetwork.png)



