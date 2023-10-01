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

![](/assets/network-virtualnet-ovs-codevsd1.png)ofproto 类结构是每一个ofproto\(ovs网桥\)实现一个

ofproto provider 是ofproto用来监控和控制OpenFlow交换机的，struct ofproto\_class in ofproto/ofproto-provider.h， 定义了一个ofproto provider实现，硬件或者软件实现。

Openvswitch有一个内置的ofproto provider叫做ofproto-dpif,建立在一个操作datapaths库之上，叫做dpif。 datapath是一个流表，只支持精确匹配的流，不支持通配符匹配。当一个数据包到达网络设备， datapath查找这张表，如果匹配，执行关联的动作，如果不匹配，datapath传递数据包到ofproto-dpif, ofproto-dpif保存着完整的OpenFlow流表。如果数据包在这张表匹配，ofproto-dpif执行动作并且插入一条新的项目到dpif 流表。（否则，ofproto-dpif传递数据包给ofproto发送数据包给OpenFlow controller，如果配置了。）

### **netdev**

Openvswitch的库lib/netdev-provider.h，实现在lib/netdev.c，是一个网络设备的抽象， Ethernet interface的抽象。

每一个交换机的端口必须实现相应的netdev操作，比如获取netdev的MTU，获取RX和TX队列数。

### netdev-provider {#24-netdev-provider}

![](/assets/network-virtualnet-ovs-codevsd2.png)

netdev provider实现特定操作系统和硬件的网络设备接口，比如ethernet设备。Openvswtich必须能够每一个交换机上的端口，所以你需要实现一个netdev provider,可以配合你的交换机硬件和软件。

所有的netdev类型：

* linux netdev\(lib/netdev-linux.c linux平台\)

  * system=netdev\__linux\_class_
  * tap - netdev\__tap\_class_
  * internal - netdev\__tap\_class_

* bsd netdev\(lib/netdev-bsd.c bsd平台\)
  * system - netdev\_bsd\_class
  * tap - netdev\_tap\_class
* windows netdev\(windows平台\)
  * system - netdev\_windows\_class
  * internal - netdev\_internal\_class
* dummy netdev\(lib/netdev\_dummy.c\)
  * dummy - dummy\_class
  * dummy-internal - dummy\_internal\_class
  * dummy-pmd - dummy\_pmd\_class
* vport netdev\(lib/netdev-vport.c, 一个vport，在datapath中保存的port引用，这样Port可以被netdev\_open\)
  * tunnel class:
    * geneve
    * gre
    * vxlan
    * lisp
    * stt
  * patch - patch\_class
* dpdk netdev
  * dpdk\_class
  * dpdk\_ring\_class
  * dpdk\_vhost\_class
  * dpdk\_vhost\_client\_class

### vswitchd 调用流程

![](/assets/network-virtualnet-ovs-ovscallflow1.png)

vswitchd的入口在vswitchd/ovs-vswitchd.c,逻辑控制图如上图

一开始，初始化bridge 模块（vswitchdbridge.c），bridge模块从ovsdb获取一些配置，然后进入main\_loop， 第一轮循环，初始化一些库，包括dpdk和ofproto。这些资源只初始化一次。

然后每一个datapath通过运行ofproto_typerun\(\)干自己的活，每一个调用具体实现的type\_run\(\)。_

然后每一个bridge调用ofproto\_run\(\)干活，每一个调用具体实现ofproto的run\(\)

然后ovs-vswitchd处理RPC（JSON\_RPC）消息，从ovs-appctl来的请求和ovsdb-server来的请求

然后，netdev\_run\(\)处理各种netdevs

上边这些做完后，bridge, unixctl server和netdev进入阻塞状态等待新的信号到来。

```
int main()
{
    /* step.1. init bridge module, obtain configs from ovsdb */
    bridge_init();

    /* step.2. deamon loop */
    while (!exiting) {
        /* step.2.1. process control messages from OpenFlow Controller and CLI */
        bridge_run()
          |
          |--dpdk_init()
          |--bridge_init_ofproto() // init bridges, only once
          |--bridge_run__()
              |
              |--for(datapath types):/* Let each datapath type do the work that it needs to do. */
                     ofproto_type_run(type)
              |--for(all_bridges):
                     ofproto_run(bridge) // handle messages from OpenFlow Controller

        unixctl_server_run(unixctl); /* receive control messages from CLI (ovs-appctl <xxx>) */
        netdev_run(); /* Performs periodic work needed by all the various kinds of netdevs */

        /* step.2.2. wait events arriving */
        bridge_wait();
        unixctl_server_wait(unixctl);
        netdev_wait();

        /* step.2.3. block util events arrive */
        poll_block();
    }
}
```

![](/assets/network-virtualnet-ovs-codevsdinternal.png)

**datapath processing**

ofproto/ofproto_dpif.c:type\__run\(\)

vswitchd循环所有datapath 类型，不同类型的datapath处理各自的逻辑。比如，处理port change，处理upcall\(发送不匹配的包给OpenFlow controller\)。

```
static int
type_run(const char *type)
{
    udpif_run(backer->udpif);
      |
      |--unixctl_command_reply() // handles upcall

    if (backer->recv_set_enable) {
        udpif_set_threads(backer->udpif, n_handlers, n_revalidators);
    }

    if (backer->need_revalidate) {
        // revalidate

        udpif_revalidate(backer->udpif);
    }

    /* Check for and handle port changes dpif. */
    process_dpif_port_changes(backer);
}
```

**bridge processing**

每次循环，vswitchd让每一个bridge处理各自的事情，在ofproto_run\(\),_ 在vswitch/bridge.c

```
int
ofproto_run(struct ofproto *p)
{
    p->ofproto_class->run(p); // calls into ofproto-dpif.c for dpif class

    if (p->ofproto_class->port_poll) {
        while ((error = p->ofproto_class->port_poll(p, &devname)) != EAGAIN) {
            process_port_change(p, error, devname);
        }
    }

    if (new_seq != p->change_seq) {
        /* Update OpenFlow port status for any port whose netdev has changed.
         *
         * Refreshing a given 'ofport' can cause an arbitrary ofport to be
         * destroyed, so it's not safe to update ports directly from the
         * HMAP_FOR_EACH loop, or even to use HMAP_FOR_EACH_SAFE.  Instead, we
         * need this two-phase approach. */
        SSET_FOR_EACH (devname, &devnames) {
            update_port(p, devname);
        }
    }

    connmgr_run(p->connmgr, handle_openflow); // handles openflow messages
}
```

在上边的代码中， bridge首先调用 ofproto class的run\(\)方法，让每种类型特定的事情，然后处理port change和OpenFlow messages。 对于dpif, run\(\)方法在ofproto/ofproto-dpif.c

```
static int
run(struct ofproto *ofproto_)
{
    if (ofproto->netflow) {
        netflow_run(ofproto->netflow);
    }
    if (ofproto->sflow) {
        dpif_sflow_run(ofproto->sflow);
    }
    if (ofproto->ipfix) {
        dpif_ipfix_run(ofproto->ipfix);
    }
    if (ofproto->change_seq != new_seq) {
        HMAP_FOR_EACH (ofport, up.hmap_node, &ofproto->up.ports) {
            port_run(ofport);
        }
    }
    if (ofproto->lacp_enabled || ofproto->has_bonded_bundles) {
        HMAP_FOR_EACH (bundle, hmap_node, &ofproto->bundles) {
            bundle_run(bundle);
        }
    }

    stp_run(ofproto);
    rstp_run(ofproto);
    if (mac_learning_run(ofproto->ml)) {
        ofproto->backer->need_revalidate = REV_MAC_LEARNING;
    }
    if (mcast_snooping_run(ofproto->ms)) {
        ofproto->backer->need_revalidate = REV_MCAST_SNOOPING;
    }
    if (ofproto->dump_seq != new_dump_seq) {
        /* Expire OpenFlow flows whose idle_timeout or hard_timeout has passed. */
        LIST_FOR_EACH_SAFE (rule, next_rule, expirable,
                            &ofproto->up.expirable) {
            rule_expire(rule_dpif_cast(rule), now);
        }

        /* All outstanding data in existing flows has been accounted, so it's a
         * good time to do bond rebalancing. */
        if (ofproto->has_bonded_bundles) {
            HMAP_FOR_EACH (bundle, hmap_node, &ofproto->bundles)
                if (bundle->bond)
                    bond_rebalance(bundle->bond);
        }
    }
}
```

**bridge\_run\(\)总结**

    void
    bridge_run(void)
    {
        /* step.1. init all needed */
        ovsdb_idl_run(idl); // handle RPC; sync with remote OVSDB
        if_notifier_run(); // TODO: not sure what's doing here

        if (cfg)
            dpdk_init(&cfg->other_config);

        /* init ofproto library.  This only runs once */
        bridge_init_ofproto(cfg);
          |
          |--ofproto_init(); // resiter `ofproto/list` command
               |
               |--ofproto_class_register(&ofproto_dpif_class) // register default ofproto class
               |--for (ofproto classes):
                    ofproto_classes[i]->init() // for ofproto_dpif_class, this will call the init() method in ofproto-dpif.c

        /* step.2. datapath & bridge processing */
        bridge_run__();
          |
          |--FOR_EACH (type, &types) /* Let each datapath type do the work that it needs to do. */
          |    ofproto_type_run(type);
          |
          |--FOR_EACH (br, &all_bridges) /* Let each bridge do the work that it needs to do. */
               ofproto_run(br->ofproto);
                   |
                   |--ofproto_class->run()
                   |--connmgr_run(connmgr, handle_openflow) // handles messages from OpenFlow controller

        /* step.3. commit to ovsdb if needed */
        ovsdb_idl_txn_commit(txn);
    }

### Unixctl IPC 处理 {#46-unixctl-ipc-handling}

在每一轮循环中ovs-vswitchd, unixctl\__server\__run\(\)只调用一次。unix server首先接收IPC连接， 处理每一个连接的请求。

```
void
unixctl_server_run(struct unixctl_server *server)
{
    // accept connections
    for (i = 0; i < 10; i++) {
        error = pstream_accept(server->listener, &stream);
        if (!error) {
            conn->rpc = jsonrpc_open(stream);
        } else if (error == EAGAIN) {
            break;
    }

    // process requests from each connection
    LIST_FOR_EACH_SAFE (conn, next, node, &server->conns) {
        run_connection(conn);
          |
          |--jsonrpc_run()
          |    |--stream_send()
          |--jsonrpc_recv(conn_rpc, &msg)
               |--assert(msg.type == JSONRPC_REQUEST)
               |--process_command(conn, msg) // format the received text to desired output
                    |--registerd unixctl command callback
    }
}
```

### netdev run

vswitchd循环所有的网络设备， 更新任何的netdev的更新

```
/* Performs periodic work needed by all the various kinds of netdevs.
 *
 * If your program opens any netdevs, it must call this function within its
 * main poll loop. */
void
netdev_run(void)
{
    netdev_initialize();

    struct netdev_registered_class *rc;
    CMAP_FOR_EACH (rc, cmap_node, &netdev_classes) {
        if (rc->class->run)
            rc->class->run(rc->class);
    }
}
```

每一类网络设备需要实现struct netdev_class的方法，比如bsd, linux,dpdk,这些实现在lib/netdev\__xxx.c

rc-&gt;class-&gt;run\(rc-&gt;class\)运行具体实现，比如linux， 会运行netdev\__linux\_run\(\)在lib/netdev\_linux.c. _这些回调函数里边检测linux network设备变更\(通过netlink\)， 比如mtu, src/dst mac变化，更新这些设备到netdev.

```
static void
netdev_linux_run(const struct netdev_class *netdev_class OVS_UNUSED)
{
    if (netdev_linux_miimon_enabled())
        netdev_linux_miimon_run();

    /* Returns a NETLINK_ROUTE socket listening for RTNLGRP_LINK,
     * RTNLGRP_IPV4_IFADDR and RTNLGRP_IPV6_IFADDR changes */
    sock = netdev_linux_notify_sock();

    do {
        error = nl_sock_recv(sock, &buf, false); // receive from kernel space
        if (!error) {
            if (rtnetlink_parse(&buf, &change)) {
                netdev_linux_update(netdev, &change);
                  |  // update netdev changes, e.g. mtu, src/dst mac, etc
                  |- netdev_linux_changed(netdev, flags, 0);
            }
        } else if (error == ENOBUFS) {
            netdev_get_devices(&netdev_linux_class, &device_shash);
            SHASH_FOR_EACH (node, &device_shash) {
                get_flags(netdev_, &flags);
                netdev_linux_changed(netdev, flags, 0);
            }
        }
    } while (!error);
}
```

## 5. event loop: wait & block {#5-event-loop-wait--block}

## 6. ingress flow diagram \(TODO\) {#6-ingress-flow-diagram-todo}

## 7. egress flow diagram \(TODO\) {#7-egress-flow-diagram-todo}





