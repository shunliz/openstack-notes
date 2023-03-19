传统3层架构![](/assets/network-design-spineleaf3.png)

spine-leaf架构

  
![](/assets/network-design-spineleaf1.png)

![](/assets/nework-design-spineleaf2.png)EBGP连接都是单跳。这样就不用依赖IGP构建nexthop网络，EBGP的nexthop都在链路的另一端。

* 采用ASN中保留给数据中心内部的ASN 64512到65534，共1023个ASN。
* 所有Super Spine共用一个唯一的ASN。
* 每组Spine共用一个唯一的ASN。
* 每个Leaf有一个唯一的ASN。



