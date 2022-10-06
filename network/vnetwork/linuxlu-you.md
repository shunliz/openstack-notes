网络包在发送的时候，需要从本机的多个网卡设备中选择一个合适的发送出去。网络包在接收的时候，也需要进行路由选择，如果是属于本设备的包就往上层送到网络层、传输层直到 socket 的接收缓存区中。如果不是本设备上的包，就选择合适的设备将其转发出去。

![](/assets/network-virtualnet-linuxnet-route1.png)

在默认情况下，Linux 只有 local 和 main 两个路由表。如果内核编译时支持策略路由，那么管理员最多可以配置  255 个独立的路由表。如果你的服务器上创建了多个网络命名空间的话，那么就会存在多套路由表。以除了默认命名网络空间外，又创了了一个新网络命名空间的情况为例，路由表在整个内核数据结构中的关联关系总结如下图所示。

![](/assets/network-virtualnet-linuxnet-route2.png)

## 如果本机有目标ip，则会直接访问本地; 如果本地没有目标ip，则看第2步

1. **用route -n查看路由，如果路由条目里包含了目标ip的网段，则数据包就会从对应路由条目后面的网卡出去**
2. **如果没有对应网段的路由条目，则全部都走网关**
3. **如果网关也没有，则报错：网络不可达**

**网关只能加路由条目里已有的路由网段里的一个IP \(ping不通此IP都可以） 加网关不需要指定子网掩码**

**永久配置\(如果机器有多张网卡,只需要一张网卡配置网关, 网关要与配置的网卡同网段\)**



qugaga [https://blog.csdn.net/weixin\_42348333/article/details/105314662](https://blog.csdn.net/weixin_42348333/article/details/105314662)

docker qugaga [https://www.freesion.com/article/4678245458/](https://www.freesion.com/article/4678245458/)

openwrt [https://openwrt.org/](https://openwrt.org/)

