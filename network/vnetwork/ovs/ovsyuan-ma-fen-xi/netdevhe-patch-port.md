### Patch port

相比veth, patch port 性能更好，优雅的ovs agent重启网络流不断。但是不能通过tcpdump进行抓包。

### ovs netdev

![](/assets/network-virtualnet-ovs-codenetdev1.png)一个网络设备（如网卡）有两部分，一部分在内核，一部分在用户空间。内核部分负责收发包，用户空间部分负责管理内核部分，比如修改MTU，开启关闭队列。内核和用户空间的通信是通过netlink和ioctl\(废弃了\)

对于虚拟机设备，比如TUN/TAP，工作原理类似。只是TAP设备的包不是来自外部，而是来自用户空间，发出的包不是发向外部，而是发给用户空间。

在ovs，一个struct netdev代表一个用户态网络设备，用来管理内核态的设备。

`lib/netdev-provider.h`

```
./* A network device (e.g. an Ethernet device) */
struct netdev {
    char *name;                         /* Name of network device. */
    const struct netdev_class *netdev_class; /* Functions to control
                                                this device. */
    ...

    int n_txq;
    int n_rxq;
    int ref_cnt;                        /* Times this devices was opened. */
};
```

netdev\_class 是一个通用的网络设备抽象，定义在`lib/netdev-provider.h`

```
struct netdev_class {
    const char *type; /* Type of netdevs in this class, e.g. "system", "tap", "gre", etc. */
    bool is_pmd;      /* If 'true' then this netdev should be polled by PMD threads. */

    /* ## Top-Level Functions ## */
    int (*init)(void);
    void (*run)(const struct netdev_class *netdev_class);
    void (*wait)(const struct netdev_class *netdev_class);

    /* ## netdev Functions ## */
    int (*construct)(struct netdev *);
    void (*destruct)(struct netdev *);

    ...

    int (*rxq_recv)(struct netdev_rxq *rx, struct dp_packet_batch *batch);
    void (*rxq_wait)(struct netdev_rxq *rx);
};
```

.一个设备类型被使用前必须实现netdev\_class的方法才能成为一个netdev provider. 有各种不同的netdev provider， BSD， windows， linux等。

![](/assets/network-virtualnet-ovs-codenetdev2.png)

上图显示了netdev和netdev\_provider在ovs中额关系。下边讨论linux netdev, vport netdev和dpdk netdev

#### Linux netdev

`struct linux_netdev`定义在`lib/netdev-linux.c`

.

linux netdev是linux系统上的网络设备，调用netdev的send\(\)方法将会把数据包从用户空间发往内核，然后会被内设备驱动处理，设备有一下三种

* system - netdev\_ _ linux \_ class: \_默认的 linux网络设备，对于物理设备来说是这样。 同时也叫做external设备，它从外部接收包，也可以把包发往外部。
* internal -  netdev_ \_internal_ _class: linux的一种特殊设备，和system类似，send（）函数不能发往外部和从外部收包，只能通过AF\_PACKET socket_发往内核。
* tap - netdev\__tap_\_class：send\(\) 通过write系统调用发送数据包到内核（）

```
const struct netdev_class netdev_linux_class =
    NETDEV_LINUX_CLASS(
        "system",
        netdev_linux_construct,
        netdev_linux_get_stats,
        netdev_linux_get_features,
        netdev_linux_get_status);

const struct netdev_class netdev_tap_class =
    NETDEV_LINUX_CLASS(
        "tap",
        netdev_linux_construct_tap,
        netdev_tap_get_stats,
        netdev_linux_get_features,
        netdev_linux_get_status);

const struct netdev_class netdev_internal_class =
    NETDEV_LINUX_CLASS(
        "internal",
        netdev_linux_construct,
        netdev_internal_get_stats,
        NULL,                  /* get_features */
        netdev_internal_get_status);
```

#### vport netdev

`lib/netdev-vport.c`

一个vport是一个ovs datapath的抽象虚拟端口， vport netdev是ovs vport的用户空间部分， 分两类tunnel和patch

* tunnel类
  * geneve
  * gre
  * vxlan
  * lisp
  * stt
* patch -  patch\_class
  * bridge&lt;-&gt;bridge连接

注册tunnel vport netdev\_vport\_tunnel\_register\(\)，注册patch vport `netdev_vport_patch_register()`

都会调用 `netdev_register_provider()`

```
void
netdev_vport_tunnel_register(void)
{
    static const struct vport_class vport_classes[] = {
        TUNNEL_CLASS("geneve", "genev_sys", netdev_geneve_build_header,
                netdev_tnl_push_udp_header, netdev_geneve_pop_header),
        TUNNEL_CLASS("gre", "gre_sys", netdev_gre_build_header,
                netdev_gre_push_header, netdev_gre_pop_header),
        TUNNEL_CLASS("vxlan", "vxlan_sys", netdev_vxlan_build_header,
                netdev_tnl_push_udp_header, netdev_vxlan_pop_header),
        TUNNEL_CLASS("lisp", "lisp_sys", NULL, NULL, NULL),
        TUNNEL_CLASS("stt", "stt_sys", NULL, NULL, NULL),
    };

    for (i = 0; i < ARRAY_SIZE(vport_classes); i++) {
        netdev_register_provider(&vport_classes[i].netdev_class);
    }
}

void
netdev_vport_patch_register(void)
{
    static const struct vport_class patch_class =
        { NULL,
            { "patch", false,
              VPORT_FUNCTIONS(get_patch_config, set_patch_config,
                              NULL, NULL, NULL, NULL, NULL) }};
    netdev_register_provider(&patch_class.netdev_class);
}
```

下边是简化的初始化宏

```
#define VPORT_FUNCTIONS(GET_CONFIG, SET_CONFIG,             \
                        GET_TUNNEL_CONFIG, GET_STATUS,      \
                        BUILD_HEADER,                       \
                        PUSH_HEADER, POP_HEADER)            \
    netdev_vport_alloc,                                     \
    netdev_vport_construct,                                 \
    BUILD_HEADER,                                           \
    PUSH_HEADER,                                            \
    POP_HEADER,                                             \
                                                            \
    NULL,                       /* send */                  \
    NULL,                       /* send_wait */             \
    ...
    NULL,                   /* rx_recv */                  \
    NULL,                   /* rx_drain */


#define TUNNEL_CLASS(NAME, DPIF_PORT, BUILD_HEADER, PUSH_HEADER, POP_HEADER)   \
    { DPIF_PORT,                                                               \
        { NAME, false,                                                         \
          VPORT_FUNCTIONS(get_tunnel_config,                                   \
                          set_tunnel_config,                                   \
                          get_netdev_tunnel_config,                            \
                          tunnel_get_status,                                   \
                          BUILD_HEADER, PUSH_HEADER, POP_HEADER) }}
```

注意所有的vport的send和rx_recv都是NULL， 意味着数据包不能通过vport从用户空间发往内核空间， vport不能从物理网卡接收数据包。 vport用于在datapath之间转包，或者通过调用内核dev\_queue_\_xmit\(\)发送包。\_

#### dpdk\_netdev

* `dpdk_class`

* `dpdk_ring_class`

* `dpdk_vhost_class`

* `dpdk_vhost_client_class`

### Patch Port

patch port是一种vport类型的netdev， 通过`netdev_register_provider(const struct netdevclass *newclass)在lib/netdev-vport.c注册。 初始化和注册了一个netdev provider.注册后可以通过netdevopen() 打开`

patch port 只接收一个peer参数，表示对端，和linux veth pair类似，是用来替换veth引入的，用来连接两个bridge。patch port不从物理设备接收数据包，只从另一端的网桥（datapath）接收数据包，OUTPUT到另外一个peer，用来连接两个bridge.

OUTPUT动作只发送数据包到其他vport，没有memcpy, 没有上下文切换，所有动作在datapath执行，不涉及内核网络协议栈。

为什么可以在host看见patch port，但是不能通过tcpdum抓包，抓包是通过插入过滤代码到内核，在链路层拷贝所有的入方向包，发送到一个buffer，用户态读取buffer获取数据包。

![](/assets/network-virtualnet-ovs-codenetdev3.png)

在LInux中，数据包的拷贝是通过netif\_rx\(\), 在linux内核代码net/core/dev.c

```
/**
 *  netif_rx    -   post buffer to the network code
 *
 *  This function receives a packet from a device driver and queues it for
 *  the upper (protocol) levels to process. It always succeeds.
 *
 *  return values:
 *  NET_RX_SUCCESS  (no congestion)
 *  NET_RX_DROP     (packet was dropped)
 */
int netif_rx(struct sk_buff *skb)
{
    trace_netif_rx_entry(skb);

    return netif_rx_internal(skb);
}
```

当数据包通过物理设备（如eth0）或者虚拟设备（如tap,tun）,设备驱动发送数据包到内核netif_rx\(skb\). trace\_netif\_rx\_entry\(skb\)过滤数据包并拷贝。_

对于patch port，驱动不会调用netif\_rx\(\)（因为数据包的目的是对端，不是内核协议栈）， 驱动没有过滤代码，所以不能拷贝包，所以不能抓包。同理为啥dpdk的设备也不能使用tcpdump, netstat等工具，不能抓包。

### Patch port的性能提升

不用额外的内核datapath查找，和到从内核空间到用户空间的路径开销。

### Internal Port

Linux bridge port是纯2层的port不能配置IP， linux网桥增加所有物理网卡后，host没有办法连接。

L2 port工作在数据面， 用来转发数据。L3 port工作在控制面，用来管理bridge。

为了保证主机仍然可以连接，两种方案

* 保留一个物理网卡做管理连接
* 使用虚拟管理口

**方案1 保留物理管理口**

![](/assets/network-virtualnet-ovs-codeinterport.png)好处：

* 数据面和控制面分离
* 稳定
* 即使linux bridge工作异常，仍然可以正常工做。

缺点：

* 资源利用率不高

**方案2 虚拟机管理口**

![](/assets/network-virtualnet-ovs-codeinterport2.png)一个虚拟的port用来连接bridge并且配置了ip，因为所有的物理口都连接了bridge， 这个虚拟机口也要连接到bridge才能通过外接访问到这个虚拟管理口。

问题来了，管理流量的出入也是通过数据面端口走，因为只有物理网卡可以和外接通信，并且物理口是数据面的port。

出主机的的数据包的MAC地址将是虚拟接口的MAC，不是物理接口的MAC，但是数据包是从物理接口出去的。

正常的物理交换机端口不应该配置IP， 配置IP后数据将不会发给交换机处理而直接在这个端口中止并由协议栈接收处理（刚好适合做容器的NIC）。

**为了解决这个问题，ovs引入了internal port**

internal port底层还是通过tap实现的。

**高级用法， internal port作为容器的NIC**

![](/assets/network-virtualnet-ovs-vportinter1.png)

使用ovs internal port连接容器到ovs

1. 获取容器namespace ns1
2. 创建ovs internal port tap1
3. 移动tap1从默认namespace到容器namespace ns1
4. 禁用ns1默认网络设备, 通常是eth0
5. 为tap1配置IP，设置为ns1默认网络设备eth0.添加路由

### ovs internal port方式接容器到ovs， 和veth pair方式接容器到ovs对比？



