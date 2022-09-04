Linux提供了许多虚拟设备，这些虚拟设备有助于构建复杂的网络拓扑，满足各种网络需求。

## 网桥（bridge） {#h2--bridge-}

网桥是一个二层设备，工作在链路层，主要是根据MAC学习来转发数据到不同的port。

```
# 创建网桥
brctl addbr br0
# 添加设备到网桥
brctl addif br0 eth1
# 查询网桥mac表
brctl showmacs br0
```

 

---

## veth {#h2-veth}

veth pair是一对虚拟网络设备，一端发送的数据会由另外一端接受，常用于不同的网络命名空间。

```
# 创建veth pair
ip link add veth0 type veth peer name veth1
# 将veth1放入另一个netns
ip link set veth1 netns newns
```

[Linux Switching – Interconnecting Namespaces](https://link.segmentfault.com/?enc=%2Bqxih9L4deigRn0ErVlPMw%3D%3D.3IskfrrBWgOaIz5pOlswvdyEKfa%2B6J8xhp9275Uf%2BS5R8PRvUWtvRdlIocn7ZyfU)

## veth设备的特点 {#item-1}

* veth和其它的网络设备都一样，一端连接的是内核协议栈。
* veth设备是成对出现的，另一端两个设备彼此相连
* 一个设备收到协议栈的数据发送请求后，会将数据发送到另一个设备上去。

下面这张关系图很清楚的说明了veth设备的特点：

```
+----------------------------------------------------------------+
|                                                                |
|       +------------------------------------------------+       |
|       |             Newwork Protocol Stack             |       |
|       +------------------------------------------------+       |
|              ↑               ↑               ↑                 |
|..............|...............|...............|.................|
|              ↓               ↓               ↓                 |
|        +----------+    +-----------+   +-----------+           |
|        |   eth0   |    |   veth0   |   |   veth1   |           |
|        +----------+    +-----------+   +-----------+           |
|192.168.1.11  ↑               ↑               ↑                 |
|              |               +---------------+                 |
|              |         192.168.2.11     192.168.2.1            |
+--------------|-------------------------------------------------+
               ↓
         Physical Network
```

上图中，我们给物理网卡eth0配置的IP为192.168.1.11， 而veth0和veth1的IP分别是192.168.2.11和192.168.2.1。

## 示例 {#item-2}

我们通过示例的方式来一步一步的看看veth设备的特点。

### 只给一个veth设备配置IP {#item-2-1}

先通过ip link命令添加veth0和veth1，然后配置veth0的IP，并将两个设备都启动起来

```
dev@debian:~$ sudo ip link add veth0 type veth peer name veth1
dev@debian:~$ sudo ip addr add 192.168.2.11/24 dev veth0
dev@debian:~$ sudo ip link set veth0 up
dev@debian:~$ sudo ip link set veth1 up
```

这里不给veth1设备配置IP的原因就是想看看在veth1没有IP的情况下，veth0收到协议栈的数据后会不会转发给veth1。

ping一下192.168.2.1，由于veth1还没配置IP，所以肯定不通

```
dev@debian:~$ ping -c 4 192.168.2.1
PING 192.168.2.1 (192.168.2.1) 56(84) bytes of data.
From 192.168.2.11 icmp_seq=1 Destination Host Unreachable
From 192.168.2.11 icmp_seq=2 Destination Host Unreachable
From 192.168.2.11 icmp_seq=3 Destination Host Unreachable
From 192.168.2.11 icmp_seq=4 Destination Host Unreachable

--- 192.168.2.1 ping statistics ---
4 packets transmitted, 0 received, +4 errors, 100% packet loss, time 3015ms
pipe 3
```

但为什么ping不通呢？是到哪一步失败的呢？

先看看抓包的情况，从下面的输出可以看出，veth0和veth1收到了同样的ARP请求包，但没有看到ARP应答包：

```
dev@debian:~$ sudo tcpdump -n -i veth0
tcpdump: verbose output suppressed, use -v or -vv 
for
 full protocol decode
listening on veth0, link-type EN10MB (Ethernet), capture size 262144 bytes
20:20:18.285230 ARP, Request who-has 192.168.2.1 tell 192.168.2.11, length 28
20:20:19.282018 ARP, Request who-has 192.168.2.1 tell 192.168.2.11, length 28
20:20:20.282038 ARP, Request who-has 192.168.2.1 tell 192.168.2.11, length 28
20:20:21.300320 ARP, Request who-has 192.168.2.1 tell 192.168.2.11, length 28
20:20:22.298783 ARP, Request who-has 192.168.2.1 tell 192.168.2.11, length 28
20:20:23.298923 ARP, Request who-has 192.168.2.1 tell 192.168.2.11, length 28

dev@debian:~$ sudo tcpdump -n -i veth1
tcpdump: verbose output suppressed, use -v or -vv 
for
 full protocol decode
listening on veth1, link-type EN10MB (Ethernet), capture size 262144 bytes
20:20:48.570459 ARP, Request who-has 192.168.2.1 tell 192.168.2.11, length 28
20:20:49.570012 ARP, Request who-has 192.168.2.1 tell 192.168.2.11, length 28
20:20:50.570023 ARP, Request who-has 192.168.2.1 tell 192.168.2.11, length 28
20:20:51.570023 ARP, Request who-has 192.168.2.1 tell 192.168.2.11, length 28
20:20:52.569988 ARP, Request who-has 192.168.2.1 tell 192.168.2.11, length 28
20:20:53.570833 ARP, Request who-has 192.168.2.1 tell 192.168.2.11, length 28
```

为什么会这样呢？了解ping背后发生的事情后就明白了：

1. ping进程构造ICMP echo请求包，并通过socket发给协议栈，
2. 协议栈根据目的IP地址和系统路由表，知道去192.168.2.1的数据包应该要由192.168.2.11口出去
3. 由于是第一次访问192.168.2.1，且目的IP和本地IP在同一个网段，所以协议栈会先发送ARP出去，询问192.168.2.1的mac地址
4. 协议栈将ARP包交给veth0，让它发出去
5. 由于veth0的另一端连的是veth1，所以ARP请求包就转发给了veth1
6. veth1收到ARP包后，转交给另一端的协议栈
7. 协议栈一看自己的设备列表，发现本地没有192.168.2.1这个IP，于是就丢弃了该ARP请求包，这就是为什么只能看到ARP请求包，看不到应答包的原因

### 给两个veth设备都配置IP {#item-2-2}

给veth1也配置上IP

```
dev@debian:~$ sudo ip addr add 192.168.2.1/24 dev veth1
```

再ping 192.168.2.1成功（由于192.168.2.1是本地IP，所以默认会走lo设备，为了避免这种情况，这里使用ping命令带上了-I参数，指定数据包走指定设备）

```
dev@debian:~$ ping -c 4 192.168.2.1 -I veth0
PING 192.168.2.1 (192.168.2.1) from 192.168.2.11 veth0: 56(84) bytes of data.
64 bytes from 192.168.2.1: icmp_seq=1 ttl=64 time=0.032 ms
64 bytes from 192.168.2.1: icmp_seq=2 ttl=64 time=0.048 ms
64 bytes from 192.168.2.1: icmp_seq=3 ttl=64 time=0.055 ms
64 bytes from 192.168.2.1: icmp_seq=4 ttl=64 time=0.050 ms

--- 192.168.2.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3002ms
rtt min/avg/max/mdev = 0.032/0.046/0.055/0.009 ms
```

> 注意：对于非debian系统，这里有可能ping不通，主要是因为内核中的一些ARP相关配置导致veth1不返回ARP应答包，如ubuntu上就会出现这种情况，解决办法如下：  
>  root@ubuntu:~\# echo 1 &gt; /proc/sys/net/ipv4/conf/veth1/accept\_local  
> root@ubuntu:~\# echo 1 &gt; /proc/sys/net/ipv4/conf/veth0/accept\_local  
> root@ubuntu:~\# echo 0 &gt; /proc/sys/net/ipv4/conf/all/rp\_filter  
> root@ubuntu:~\# echo 0 &gt; /proc/sys/net/ipv4/conf/veth0/rp\_filter  
> root@ubuntu:~\# echo 0 &gt; /proc/sys/net/ipv4/conf/veth1/rp\_filter

再来看看抓包情况，我们在veth0和veth1上都看到了ICMP echo的请求包，但为什么没有应答包呢？上面不是显示ping进程已经成功收到了应答包吗？

```
dev@debian:~$ sudo tcpdump -n -i veth0
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on veth0, link-type EN10MB (Ethernet), capture size 262144 bytes
20:23:43.113062 IP 192.168.2.11 > 192.168.2.1: ICMP echo request, id 24169, seq 1, length 64
20:23:44.112078 IP 192.168.2.11 
> 192.168.2.1: ICMP echo request, id 24169, seq 2, length 64
20:23:45.111091 IP 192.168.2.11 > 192.168.2.1: ICMP echo request, id 24169, seq 3, length 64
20:23:46.110082 IP 192.168.2.11 > 192.168.2.1: ICMP echo request, id 24169, seq 4, length 64


dev@debian:~$ sudo tcpdump -n -i veth1
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on veth1, link-type EN10MB (Ethernet), capture size 262144 bytes
20:24:12.221372 IP 192.168.2.11 > 192.168.2.1: ICMP echo request, id 24174, seq 1, length 64
20:24:13.222089 IP 192.168.2.11 > 192.168.2.1: ICMP echo request, id 24174, seq 2, length 64
20:24:14.224836 IP 192.168.2.11 > 192.168.2.1: ICMP echo request, id 24174, seq 3, length 64
20:24:15.223826 IP 192.168.2.11 > 192.168.2.1: ICMP echo request, id 24174, seq 4, length 64
```

看看数据包的流程就明白了：

1. ping进程构造ICMP echo请求包，并通过socket发给协议栈，
2. 由于ping程序指定了走veth0，并且本地ARP缓存里面已经有了相关记录，所以不用再发送ARP出去，协议栈就直接将该数据包交给了veth0
3. 由于veth0的另一端连的是veth1，所以ICMP echo请求包就转发给了veth1
4. veth1收到ICMP echo请求包后，转交给另一端的协议栈
5. 协议栈一看自己的设备列表，发现本地有192.168.2.1这个IP，于是构造ICMP echo应答包，准备返回
6. 协议栈查看自己的路由表，发现回给192.168.2.11的数据包应该走lo口，于是将应答包交给lo设备
7. lo接到协议栈的应答包后，啥都没干，转手又把数据包还给了协议栈（相当于协议栈通过发送流程把数据包给lo，然后lo再将数据包交给协议栈的接收流程）
8. 协议栈收到应答包后，发现有socket需要该包，于是交给了相应的socket
9. 这个socket正好是ping进程创建的socket，于是ping进程收到了应答包

抓一下lo设备上的数据，发现应答包确实是从lo口回来的：

```
dev@debian:~$ sudo tcpdump -n -i lo
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on lo, link-type EN10MB (Ethernet), capture size 262144 bytes
20:25:49.590273 IP 192.168.2.1 > 192.168.2.11: ICMP echo reply, id 24177, seq 1, length 64
20:25:50.590018 IP 192.168.2.1 > 192.168.2.11: ICMP echo reply, id 24177, seq 2, length 64
20:25:51.590027 IP 192.168.2.1 > 192.168.2.11: ICMP echo reply, id 24177, seq 3, length 64
20:25:52.590030 IP 192.168.2.1 > 192.168.2.11: ICMP echo reply, id 24177, seq 4, length 64
```

### 试着ping下其它的IP {#item-2-3}

ping 192.168.2.0/24网段的其它IP失败，ping一个公网的IP也失败：

```
dev@debian:~$ ping -c 1 -I veth0 192.168.2.2
PING 192.168.2.2 (192.168.2.2) from 192.168.2.11 veth0: 56(84) bytes of data.
From 192.168.2.11 icmp_seq=1 Destination Host Unreachable

--- 192.168.2.2 ping statistics ---
1 packets transmitted, 0 received, +1 errors, 100% packet loss, time 0ms

dev@debian:~$ ping -c 1 -I veth0 baidu.com
PING baidu.com (111.13.101.208) from 192.168.2.11 veth0: 56(84) bytes of data.
From 192.168.2.11 icmp_seq=1 Destination Host Unreachable

--- baidu.com ping statistics ---
1 packets transmitted, 0 received, +1 errors, 100% packet loss, time 0ms
```

从抓包来看，和上面第一种veth1没有配置IP的情况是一样的，ARP请求没人处理

```
dev@debian:~$ sudo tcpdump -i veth1
tcpdump: verbose output suppressed, use -v or -vv 
for
 full protocol decode
listening on veth1, link-type EN10MB (Ethernet), capture size 262144 bytes
02:25:23.223947 ARP, Request who-has 192.168.2.2 tell 192.168.2.11, length 28
02:25:24.224352 ARP, Request who-has 192.168.2.2 tell 192.168.2.11, length 28
02:25:25.223471 ARP, Request who-has 192.168.2.2 tell 192.168.2.11, length 28
02:25:27.946539 ARP, Request who-has 123.125.114.144 tell 192.168.2.11, length 28
02:25:28.946633 ARP, Request who-has 123.125.114.144 tell 192.168.2.11, length 28
02:25:29.948055 ARP, Request who-has 123.125.114.144 tell 192.168.2.11, length 28
```

---

## TAP/TUN {#h2-tap-tun}

TAP/TUN设备是一种让用户态程序向内核协议栈注入数据的设备，TAP等同于一个以太网设备，工作在二层；而TUN则是一个虚拟点对点设备，工作在三层。

```
ip tuntap add tap0 mode tap
ip tuntap add tun0 mode tun
```

```
+----------------------------------------------------------------+
|                                                                |
|  +--------------------+      +--------------------+            |
|  | User Application A |      | User Application B |<-----+     |
|  +--------------------+      +--------------------+      |     |
|               | 1                    | 5                 |     |
|...............|......................|...................|.....|
|               ↓                      ↓                   |     |
|         +----------+           +----------+              |     |
|         | socket A |           | socket B |              |     |
|         +----------+           +----------+              |     |
|                 | 2               | 6                    |     |
|.................|.................|......................|.....|
|                 ↓                 ↓                      |     |
|             +------------------------+                 4 |     |
|             | Newwork Protocol Stack |                   |     |
|             +------------------------+                   |     |
|                | 7                 | 3                   |     |
|................|...................|.....................|.....|
|                ↓                   ↓                     |     |
|        +----------------+    +----------------+          |     |
|        |      eth0      |    |      tun0      |          |     |
|        +----------------+    +----------------+          |     |
|    10.32.0.11  |                   |   192.168.3.11      |     |
|                | 8                 +---------------------+     |
|                |                                               |
+----------------|-----------------------------------------------+
                 ↓
         Physical Network
```

上图中有两个应用程序A和B，都在用户层，而其它的socket、协议栈（Newwork Protocol Stack）和网络设备（eth0和tun0）部分都在内核层，其实socket是协议栈的一部分，这里分开来的目的是为了看的更直观。

tun0是一个Tun/Tap虚拟设备，从上图中可以看出它和物理设备eth0的差别，它们的一端虽然都连着协议栈，但另一端不一样，eth0的另一端是物理网络，这个物理网络可能就是一个交换机，而tun0的另一端是一个用户层的程序，协议栈发给tun0的数据包能被这个应用程序读取到，并且应用程序能直接向tun0写数据。

这里假设eth0配置的IP是10.32.0.11，而tun0配置的IP是192.168.3.11.

> 这里列举的是一个典型的tun/tap设备的应用场景，发到192.168.3.0/24网络的数据通过程序B这个隧道，利用10.32.0.11发到远端网络的10.33.0.1，再由10.33.0.1转发给相应的设备，从而实现VPN。

下面来看看数据包的流程：

1. 应用程序A是一个普通的程序，通过socket A发送了一个数据包，假设这个数据包的目的IP地址是192.168.3.1

2. socket将这个数据包丢给协议栈

3. 协议栈根据数据包的目的IP地址，匹配本地路由规则，知道这个数据包应该由tun0出去，于是将数据包交给tun0

4. tun0收到数据包之后，发现另一端被进程B打开了，于是将数据包丢给了进程B

5. 进程B收到数据包之后，做一些跟业务相关的处理，然后构造一个新的数据包，将原来的数据包嵌入在新的数据包中，最后通过socket B将数据包转发出去，这时候新数据包的源地址变成了eth0的地址，而目的IP地址变成了一个其它的地址，比如是10.33.0.1.

6. socket B将数据包丢给协议栈

7. 协议栈根据本地路由，发现这个数据包应该要通过eth0发送出去，于是将数据包交给eth0

8. eth0通过物理网络将数据包发送出去10.33.0.1收到数据包之后，会打开数据包，读取里面的原始数据包，并转发给本地的192.168.3.1，然后等收到192.168.3.1的应答后，再构造新的应答包，并将原始应答包封装在里面，再由原路径返回给应用程序B，应用程序B取出里面的原始应答包，最后返回给应用程序A

> 这里不讨论Tun/Tap设备tun0是怎么和用户层的进程B进行通信的，对于Linux内核来说，有很多种办法来让内核空间和用户空间的进程交换数据。

从上面的流程中可以看出，数据包选择走哪个网络设备完全由路由表控制，所以如果我们想让某些网络流量走应用程序B的转发流程，就需要配置路由表让这部分数据走tun0。

### tun/tap设备有什么用？

从上面介绍过的流程可以看出来，**tun/tap设备的用处是将协议栈中的部分数据包转发给用户空间的应用程序，给用户空间的程序一个处理数据包的机会。于是比较常用的数据压缩，加密等功能就可以在应用程序B里面做进去，tun/tap设备最常用的场景是VPN，包括tunnel以及应用层的IPSec**等，比较有名的项目是[VTun](https://link.segmentfault.com/?enc=gjesZK%2BZXNxSddGWDz%2F0fQ%3D%3D.bIHua%2FbwlJGfcQGQcErZccX7HAnfOXAIAcn50Z4C4GM%3D)，有兴趣可以去了解一下。

### tun和tap的区别

**用户层程序通过tun设备只能读写IP数据包，而通过tap设备能读写链路层数据包，类似于普通socket和raw socket的差别一样，处理数据包的格式不一样。**

### 示例

**示例程序**

这里写了一个程序，它收到tun设备的数据包之后，只打印出收到了多少字节的数据包，其它的什么都不做，如何编程请参考后面的参考链接。

```
#include <net/if.h>
#include <sys/ioctl.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <string.h>
#include <sys/types.h>
#include <linux/if_tun.h>
#include<stdlib.h>
#include<stdio.h>

int tun_alloc(int flags)
{

    struct ifreq ifr;
    int fd, err;
    char *clonedev = "/dev/net/tun";

    if ((fd = open(clonedev, O_RDWR)) < 0) {
        return fd;
    }

    memset(&ifr, 0, sizeof(ifr));
    ifr.ifr_flags = flags;

    if ((err = ioctl(fd, TUNSETIFF, (void *) &ifr)) < 0) {
        close(fd);
        return err;
    }

    printf("Open tun/tap device: %s for reading...\n", ifr.ifr_name);

    return fd;
}

int main()
{

    int tun_fd, nread;
    char buffer[1500];

    /* Flags: IFF_TUN   - TUN device (no Ethernet headers)
     *        IFF_TAP   - TAP device
     *        IFF_NO_PI - Do not provide packet information
     */
    tun_fd = tun_alloc(IFF_TUN | IFF_NO_PI);

    if (tun_fd < 0) {
        perror("Allocating interface");
        exit(1);
    }

    while (1) {
        nread = read(tun_fd, buffer, sizeof(buffer));
        if (nread < 0) {
            perror("Reading from interface");
            close(tun_fd);
            exit(1);
        }

        printf("Read %d bytes from tun/tap device\n", nread);
    }
    return 0;
}
```

### 演示

```
#--------------------------第一个shell窗口----------------------
#将上面的程序保存成tun.c，然后编译
dev@debian:~$ gcc tun.c -o tun

#启动tun程序，程序会创建一个新的tun设备，
#程序会阻塞在这里，等着数据包过来
dev@debian:~$ sudo ./tun
Open tun/tap device tun1 for reading...
Read 84 bytes from tun/tap device
Read 84 bytes from tun/tap device
Read 84 bytes from tun/tap device
Read 84 bytes from tun/tap device

#--------------------------第二个shell窗口----------------------
#启动抓包程序，抓经过tun1的包
# tcpdump -i tun1
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on tun1, link-type RAW (Raw IP), capture size 262144 bytes
19:57:13.473101 IP 192.168.3.11 > 192.168.3.12: ICMP echo request, id 24028, seq 1, length 64
19:57:14.480362 IP 192.168.3.11 > 192.168.3.12: ICMP echo request, id 24028, seq 2, length 64
19:57:15.488246 IP 192.168.3.11 > 192.168.3.12: ICMP echo request, id 24028, seq 3, length 64
19:57:16.496241 IP 192.168.3.11 > 192.168.3.12: ICMP echo request, id 24028, seq 4, length 64

#--------------------------第三个shell窗口----------------------
#./tun启动之后，通过ip link命令就会发现系统多了一个tun设备，
#在我的测试环境中，多出来的设备名称叫tun1，在你的环境中可能叫tun0
#新的设备没有ip，我们先给tun1配上IP地址
dev@debian:~$ sudo ip addr add 192.168.3.11/24 dev tun1

#默认情况下，tun1没有起来，用下面的命令将tun1启动起来
dev@debian:~$ sudo ip link set tun1 up

#尝试ping一下192.168.3.0/24网段的IP，
#根据默认路由，该数据包会走tun1设备，
#由于我们的程序中收到数据包后，啥都没干，相当于把数据包丢弃了，
#所以这里的ping根本收不到返回包，
#但在前两个窗口中可以看到这里发出去的四个icmp echo请求包，
#说明数据包正确的发送到了应用程序里面，只是应用程序没有处理该包
dev@debian:~$ ping -c 4 192.168.3.12
PING 192.168.3.12 (192.168.3.12) 56(84) bytes of data.

--- 192.168.3.12 ping statistics ---
4 packets transmitted, 0 received, 100% packet loss, time 3023ms
```

### **Tun设备客户端**

```
  ...
  /* tunclient.c */

  char tun_name[IFNAMSIZ];

  /* Connect to the device */
  strcpy(tun_name, "tun77");
  tun_fd = tun_alloc(tun_name, IFF_TUN | IFF_NO_PI);  /* tun interface */

  if(tun_fd < 0){
    perror("Allocating interface");
    exit(1);
  }

  /* Now read data coming from the kernel */
  while(1) {
    /* Note that "buffer" should be at least the MTU size of the interface, eg 1500 bytes */
    nread = read(tun_fd,buffer,sizeof(buffer));
    if(nread < 0) {
      perror("Reading from interface");
      close(tun_fd);
      exit(1);
    }

    /* Do whatever with the data */
    printf("Read %d bytes from device %s\n", nread, tun_name);
  }

  ...
```

[https://backreference.org/2010/03/26/tuntap-interface-tutorial/](https://backreference.org/2010/03/26/tuntap-interface-tutorial/)

[https://www.kernel.org/doc/Documentation/networking/tuntap.txt](https://www.kernel.org/doc/Documentation/networking/tuntap.txt)

