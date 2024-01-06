# [BGP](https://so.csdn.net/so/search?q=BGP&spm=1001.2101.3001.7020)\(边界网关协议\)-----BGP路由属性

# ![](/assets/network-basic-routing-bgpattr1.png)![](/assets/network-basic-routing-bgpattr2.png)![](/assets/network-basic-routing-bgpattr3.png)

![](/assets/network-basic-routing-bgpattr4.png)![](/assets/network-basic-routing-bgpattr5.png)![](/assets/network-basic-routing-bgpattr7.png)![](/assets/network-basic-routing-bgpattr8.png)这两个属性是用于BGP路由反射器RR，防止环路用的，但功能不一样：



originator-id：防止集群内出现环路



当RR收到客户或是非客户的路由信息放射给他的其它客户时加上originator-id属性，一般是对端的BGP router-id. 当路由器收到是originator-id是自己的话就把路由信息给丢弃来达到防止环路的目的。



originator\_id属性只有当RR从客户端学到路由信息向其它客户端反射路由时才会加上，来防止环路。



cluster-list：防止集群内出现环路



属性有点类似于AS-PATH属性，它在存在路由放射组的时候用。当两台RR互为客户时，当一台RR向另外一台RR放射路由时会加上cluster-list属性，一般是自己的cluster id号来填充。如果RR收到路由信息的cluster-list属性与自己的cluster id一致的话，就把此路由信息丢弃，来达到防止环路的目的。



**AS\_PATH细节剖析**

![](/assets/network-basic-routing-bgpattraspath1.png)![](/assets/network-basic-routing-bgpattraspath2.png)![](/assets/network-basic-routing-bgpattraspath3.png)![](/assets/network-basic-routing-bgpattraspath4.png)![](/assets/network-basic-routing-bgpattraspath6.png)![](/assets/network-basic-routing-bgpattraspath9.png)![](/assets/network-basic-routing-bgpattraspath10.png)**Next\_Hop细节剖析**

![](/assets/network-basic-routing-bgpattrnh1.png)![](/assets/netowrk-basic-routing-bgpattrnh2.png)![](/assets/network-basic-routing-bgpattrnh3.png)![](/assets/network-basci-routing-bgpattrnh4.png)![](/assets/network-basic-routing-bgpattrnh5.png)![](/assets/network-basic-routing-bgpattrnh7.png)

