# vswitchd概览![](/assets/network-virtualnet-ovs-vscode1.png)

1，系统核心部件

和外界通信使用openflow

和ovsdb-server通信，用户ovsdb-protocol

和内核通信通过netlink

和系统通信通过netdev抽象接口

```
                       +-------------------+
                       |    ovs-vswitchd   |<-->ovsdb-server
                       +-------------------+
                       |      ofproto      |<-->OpenFlow controllers
                       +--------+-+--------+
                       | netdev | | ofproto|
                       +--------+ |provider|
                       | netdev | +--------+
                       |provider|
                       +--------+
```

**关键数据结构**

```
                     _
                         |   +-------------------+
                         |   |    ovs-vswitchd   |<-->ovsdb-server
                         |   +-------------------+
                         |   |      ofproto      |<-->OpenFlow controllers
                         |   +--------+-+--------+  _
                         |   | netdev | |ofproto-|   |
               userspace |   +--------+ |  dpif  |   |
                         |   | netdev | +--------+   |
                         |   |provider| |  dpif  |   |
                         |   +---||---+ +--------+   |
                         |       ||     |  dpif  |   | implementation of
                         |       ||     |provider|   | ofproto provider
                         |_      ||     +---||---+   |
                                 ||         ||       |
                          _  +---||-----+---||---+   |
                         |   |          |datapath|   |
                  kernel |   |          +--------+  _|
                         |   |                   |
                         |_  +--------||---------+
                                      ||
                                   physical
                                      NIC
                                   OVS内部架构
```

一个ovs桥控制两类资源：

* 转发面的datapath
* 绑定到它的（物理和虚拟）网络设备

关键数据结构

* 网桥实现：ofproto, ofproto-provider
* datapath管理： dpif, dpif-provider
* 网络设备管理；netdev, netdev-provider



**ofproto**

OpenFlow交换机抽象

数据结构 \(`ofproto/ofproto-provider.h`\):

* `struct ofproto`: 代表一个OpenFlow交换机\(ovs bridge\), 所有 flow/port 操作在 ofproto
* `struct ofport`: 代表一个有OpenFlow交换机的port
* `struct rule`: 代表没有关联交换机的flow
* `struct ofgroup`: 代表 OpenFlow 1.1+ 组和一个OpenFlow交换机

```
/* An OpenFlow switch. */
struct ofproto {
    const struct ofproto_class *ofproto_class;
    char *type;                 /* Datapath type. */
    char *name;                 /* Datapath name. */

    /* Settings. */
    uint64_t fallback_dpid;     /* Datapath ID if no better choice found. */
    uint64_t datapath_id;       /* Datapath ID. */

    /* Datapath. */
    struct hmap ports;          /* Contains "struct ofport"s. */
    struct simap ofp_requests;  /* OpenFlow port number requests. */
    uint16_t max_ports;         /* Max possible OpenFlow port num, plus one. */

    /* Flow tables. */
    struct oftable *tables;

    /* Rules indexed on their cookie values, in all flow tables. */

    /* List of expirable flows, in all flow tables. */

    /* OpenFlow connections. */

    /* Groups. */

    /* Tunnel TLV mapping table. */
};


/* An OpenFlow port within a "struct ofproto".
 *
 * The port's name is netdev_get_name(port->netdev).
 */
struct ofport {
    struct hmap_node hmap_node; /* In struct ofproto's "ports" hmap. */
    struct ofproto *ofproto;    /* The ofproto that contains this port. */
    struct netdev *netdev;
    struct ofputil_phy_port pp;
    ofp_port_t ofp_port;        /* OpenFlow port number. */
    uint64_t change_seq;
    long long int created;      /* Time created, in msec. */
    int mtu;
};
```

### ofproto-provider {#22-ofproto-provider}

![](/assets/network-virtualnet-ovs-codevsd1.png)



