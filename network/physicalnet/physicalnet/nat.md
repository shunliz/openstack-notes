1  NAT

![](/assets/network-basic-nat1.png)

如图所示：11.0.25.33的主机，在NAT Server的后面，左侧三台主机，是公网122.35.10.11映射出来的。



由于IPv4不够，并且出于网络完全的原因，所以出现了NAT Server。常见的，个人电脑的ip查询，总是192.168.x.x 这就是nat之后的内网地址。而在浏览器直接输入ip，获得的是外网地址。



### 1.1 NAT 种类 {#1.1%20NAT%20%E7%A7%8D%E7%B1%BB}

![](/assets/network-basic-nat2.png)

![](/assets/network-basic-nat3.png)

内网的 X=ip, y=port, 映射后，得到外网的 A=ip， b=port.

X y, 向M P S 发送请求，M P S就知道了A b，通过A b， 就可以与Xy进行通信了。这是最好穿越的完全锥形NAT。安全性差。

![](/assets/network-basic-nat4.png)

X=ip, y=port, 映射后，A=ip， b=port. 还有将要请求的主机ip=P。 五元组。



先有内网向外网发送一个请求。X首先向P发送消息，记录保留P，然后P的不同端口才可以向X发送消息。



其他主机的IP地址，不能通过，因为内网没有向他们发送请求，没有记录M和S，所以M S 都不能跟X连接。

![](/assets/network-basic-nat5.png)

X=ip, y=port, 映射后，A=ip， b=port. 还有将要请求的主机ip=P，port=q。 六元组。



先有内网向外网发送一个请求。X首先向Pq发送消息，记录保留Pq，然后P的q端口才可以向X发送消息，其他端口不可以。



其他主机的IP地址，不能通过，因为内网没有向他们发送请求，没有记录M和S，所以M S 都不能跟X连接。

![](/assets/network-basic-nat66.png)

X=ip, y=port, 映射后，A=ip， b=port. 还有将要请求的主机ip=P，port=q。 六元组。



先有内网向外网发送一个请求。X首先向Pq发送消息，记录保留Pq，然后P的q端口才可以向X发送消息，P的他端口不可以。且P只能向Ab发送数据，虽然Cd也是Xy映射出来的，但是并不接受Pq的数据。Cd是为Mn服务的。

![](/assets/network-basic-nat7.png)

![](/assets/network-basic-nat8.png)

## 2 STURN {#2%20STURN}

简单记录一下

![](/assets/network-basic-nat9.png)

## 3 TURN {#3%20TURN}

![](/assets/network-basic-nat10.png)

![](/assets/network-basic-nat11.png)

（1）--------UDP/TCP/加密UDP-----------（2）--------------UDP-----------（3）



TURN Client在NAT之后，内网地址 10.1.1.5:49721



TURN Client的映射地址（出口地址）（外网）：192.0.2.1:7000



TURN Server 地址：192.0.2.15:3478，3478一般是固定的接收请求的端口。TURN Server端看到的TURN Client端的地址，就是192.0.2.1:7000。



TURN Server接收到TURN Client请求的时候，会开辟一个中转的端口：192.0.2.15:50000。用来提供UDP中转的服务。



Peer B 与TURN Server端是同一个网段，所以，两者建立连接后，可以直接互相发送数据，通过50000端口和49191端口。当50000端口收到数据，经过内部中转给3478端口，然后3478端口再中继给TURN Client端。逆向也是这样的。



在NAT之后的Peer-A, Peer-A 先穿越NAT，得到192.0.2.150:32102， 然后发送到TURN Server 端50000端口，然后TURN Server内部中转到3478端口，3478端口再传给7000端口，TURN Client就能接收到数据。TURN Client发送数据，也是经过NAT穿越，3478端口中转50000端口，向32102端口发送数据，使得TURN Client端与Peer-A进行通信。

![](/assets/network-basic-nat12.png)

![](/assets/network-basic-nat13.png)

## **4 ICE**





STUN Server可以有两个，也可以有一个。STUN Server用于NAT穿越，也就是获得外网ip和port。



Relay server 就是上面图中的TURN server。Relay server也具有STUN Server的功能。可以映射公网IP,也有中继的功能。



 ICE目的：就是让Peer-A获取到，所有可以连接到Peer-B的通路。



通路一共有多少种：



本机网卡一个IP，局域网内，这就是一个通路了。

两端都穿越NAT，通过STUN server服务获取NAT映射后的地址，外网地址，就可以尝试P2P通信了。

P2P不通，就得通过Relay server进行通信。

如果本机有多个网卡，还会有其他路线。

如果本机有VPN，还会有VPN通信

还可能是虚拟IP

...

ICE 具体做哪些：



将所有的通路都收集起来。

对Candidate Pair排序

分别进行连通性检测

