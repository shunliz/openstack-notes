接前面这个写起，比较过linux kernel vxlan device和ovs vxlan的性能，很好奇ovs vxlan是怎么实现的，linux kernel vxlan device是用如下命令创建的。

```
ip link add vxlan0 type vxlan id 1111 dstport 5799 remote 10.145.69.49 local 10.145.69.26 dev eth4
[root@openstack607 huiwei]#  ip -d link show vxlan0
375: vxlan0: 
<
BROADCAST,MULTICAST,UP,LOWER_UP
>
 mtu 1450 qdisc noqueue state UNKNOWN mode DEFAULT qlen 1000
    link/ether ba:6f:38:6f:bf:9a brd ff:ff:ff:ff:ff:ff promiscuity 0
    vxlan id 1111 remote 10.145.69.49 local 10.145.69.26 dev eth4 srcport 0 0 dstport 5799 ageing 300

```

发现ovs也创建了一个vxlan device

```
[root@openstack607 huiwei]# ip -d link show vxlan_sys_4789
241: vxlan_sys_4789: 
<
BROADCAST,MULTICAST,UP,LOWER_UP
>
 mtu 65000 qdisc noqueue master ovs-system state UNKNOWN mode DEFAULT qlen 1000
    link/ether 56:bc:a6:9b:c2:fd brd ff:ff:ff:ff:ff:ff promiscuity 1
    vxlan id 0 srcport 0 0 dstport 4789 nolearning ageing 300 udp6zerocsumrx
    openvswitch_slave

```

ovsdb-server存储数据，ovs-vsctl通过ovsdb协议读写ovsdb-server，ovs-vswitchd进行报文转发，ovs-ofctl通过openflow信息给ovs-vswitchd添加/删除flow，那ovs-vsctl添加一个vxlan port，ovs-vswitchd怎么知道的，答案就是ovs-vswitch和ovsdb-server之间也有连接，ovs-vswitch通过IDL感知ovsdb-server的变化。

ovs-vswitchd main线程一直while处理ovsdb-server变化和ovs-appctl命令，ovs-vsctl创建一个ovs vxlan port，最终调用到了dpif\_netlink\_port\_add。

```
main
 ├─unixctl_server_create
 ├─bridge_init
 └─while//特别注意这个while循环
    ├─bridge_run
    |  ├─ovsdb_idl_run
    |  └─bridge_reconfigure
    |     └─bridge_add_ports
    |        └─iface_create
    |           └─ofproto_port_add
    |              └─(ofproto_class->port_add)port_add
    |                 └─dpif_port_add
    |                     └─(dpif_class->port_add)dpif_netlink_port_add
    ├─unixctl_server_run
    |   └─run_connection
    |      └─process_command
    |         └─(command->cb)
    ├─netdev_run
    ├─memory_wait
    ├─bridge_wait
    ├─unixctl_server_wait
    ├─netdev_wait
    └─pool_block

```

dpif\_netlink\_port\_add用netlink发送给内核，是不是和ip link add道理一样。

```
dpif_netlink_port_add
  └─dpif_netlink_rtnl_port_create_and_add
      └─dpif_netlink_rtnl_port_create
          ├─dpif_netlink_rtnl_create
          |   └─dpif_netlink_rtnl_create//注意这个RTM_NEWLINK和IFF_UP
          └─dpif_netlink_port_add__//通知内核ovs添加vxlan_vport

```

内核ovs模块初始化，内核处理ovs route netlink发送来的添加vxlan口的消息

```
vxlan_init_module
  └─rtnl_link_register(&vxlan_link_ops)
rtnl_newlink
  ├─rtnl_create_link
  |   ├─dev=kzalloc()
  |   └─alloc_netdev_mqs
  |       └─vxlan_setup
  ├─(ops->newlink)vxlan_newlink
  └─rtnl_configure_link
      └─__dev_change_flags
          └─__dev_open
              └─(ops->ndo_open)vxlan_open

```

内核除了创建一个vxlan device还要创建一个vport device，vport device里嵌套 vxlan device，内核openvswitch模块处理添加vport, 如果内核不支持RTM\_NEWLINK，这儿还可以创建出vxlan device并且UP它。

```
ovs_vport_cmd_new
  └─new_vport
      └─ovs_vport_add
           └─vxlan_create
               ├─vxlan_tnl_create
               |   ├─vxlan_dev_create//等价于ip link add vxlan0 type vxlan
               |   └─dev_change_flags
               |       └─__dev_change_flags//UP这个口
               |           └─__dev_open
               |               ├─(ops->ndo_open)也就是vxlan_open
               |               └─netpoll_rx_enable
               └─ovs_netdev_link
                   ├─vport->dev = dev_get_by_name(ovs_dp_get_net(vport->dp), name);
                   ├─netdev_rx_handler_register(vport->dev, netdev_frame_hook,vport);
                   ├─dev_disable_lro
                   └─dev_set_promiscuity

```

最终都调用到这个三个函数vxlan\_netlink/vxlan\_setup/vxlan\_open

```
vxlan_newlink
  └─vxlan_dev_configure
      ├─vxlan_ether_setup
      ├─__vxlan_change_mtu
      └─register_netdevice

vxlan_setup
  └─gro_cells_init
      ├─netif_napi_add
      └─napi_enable

vxlan_open
  └─vxlan_sock_add
     └─__vxlan_sock_add
          ├─vxlan_socket_create
          |   └─vxlan_create_sock
          |      └─udp_sock_create                                                     
          ├─udp_tunnel_notify_add_rx_port
          ├─setup_udp_tunnel_sock
          └─udp_tunnel_encap_enable

```

现在一个报文来到了物理网卡，收包，vxlan decap，查流表，然后交给虚拟机。

看这儿有多少次softirq就知道性能怎样了，一个物理网卡pps大时softirq就承受不住了。

```
#物理网卡处理中断，触发softirq
i40e_intr
  └─napi_schedule_irqoff

#i40e softirq把包放到per cpu backlog，触发softirq
i40e_napi_poll
  └─i40e_clean_rx_irq
      ├─i40e_fetch_rx_buffer
      |   └─__napi_alloc_skb
      |       └─skb->dev = napi->dev;
      ├─i40e_process_skb_fields
      └─i40e_receive_skb
          └─napi_gro_receive
              └─napi_skb_finish
                  └─netif_receive_skb_internal
                      └─__netif_receive_skb

#rx softirq处理，一起捅到udp层，把skb上的dev切换成vxlan device，再入backlog，触发softirq
process_backlog
  └─__netif_receive_skb
      └─__netif_receive_skb_core
         └─deliver_skb
            └─(pt_prev->func)ip_rcv
              └─ip_rcv_finish
                  └─ip_route_input_noref
                      └─ip_route_input_slow
                         └─ip_local_deliver
                             └─ip_local_deliver_finish
                                └─(ipprot->handler)udp_rcv
                                    └─__udp4_lib_rcv
                                       └─udp_queue_rcv_skb
                                           └─vxlan_rcv
                                              ├─skb->dev = vxlan->dev;//这儿skb的切换到vxlan device
                                              └─gro_cells_receive
                                                 └─netif_rx
                                                     └─netif_rx_internal
                                                         └─enqueue_to_backlog
                                                            └─____napi_schedule

#softirq处理，终于走到查流表了，查完流表output到qvo口，走veth pair，再到qbr，
#qbr再wakeup vhost worker，到此这个softirq的活终于干完了。
#接下来就是vhost worker把skb copy到virtio，再通知kvm irq inject。
process_backlog
  └─__netif_receive_skb
    └─__netif_receive_skb_core
       └─(rx_handler)netdev_frame_hook
           └─netdev_port_receive
               └─ovs_vport_receive
                   ├─ovs_flow_key_extract
                   └─ovs_dp_process_packet

```

再看发包，包从虚拟机来，进入ovs查流表，然后从vxlan port出去。

```
#不管怎样给datapath安装流表时需要把vxlan的local_ip和remote_ip下发给datapath   
ovs_flow_cmd_set
   └─sw_flow_actions─┐
      ovs_flow_cmd_new─┤
ovs_packet_cmd_execute─┤
                        └─ovs_nla_copy_actions
                            └─__ovs_nla_copy_actions
                               └─validate_set
                                   └─case OVS_KEY_ATTR_TUNNEL:validate_and_copy_set_tun//填好ovs_tunnel_info
                                       └─__add_action(OVS_KEY_ATTR_TUNNEL_INFO)

```

把tunnel信息加入skb，按道理dst\_entry在ip lookup后找到，难道这儿类似于dst\_entry的缓存???

```
do_execute_actions
  └─case OVS_ACTION_ATTR_SET:execute_set_action
      └─skb_dst_set//把ovs_tunnel_info加到skb上

```

datapath执行output，vport device中嵌套的vxlan device在这儿就发挥作用了。

```
do_execute_actions
  └─do_output
      └─ovs_vport_send
          ├─skb->dev = vport->dev;//没有这就不知道用哪个ndo_start_xmit
          └─dev_queue_xmit
              └─__dev_queue_xmit
                 ├─validate_xmit_skb
                 ├─dev_hard_start_xmit
                 |   └─xmit_one
                 |       └─netdev_start_xmit
                 |           └─ndo_start_xmit就是vxlan_xmit
                 └─dev_xmit_complete

```

vxlan encap，然后一口气把包从物理网卡发送出去。

```
vxlan_xmit
  └─vxlan_xmit_one
      ├─vxlan_build_skb
      └─udp_tunnel_xmit_skb
          └─iptunnel_xmit
             └─ip_local_out_sk
                └─dst_output_sk
                    └─ip_output
                        └─ip_finish_output
                            └─dst_neigh_output
                                └─neigh_direct_output
                                    └─dev_queue_xmit
                                         └─dev_hard_start_xmit
                                             └─netdev_start_xmit
                                                 └─i40e_lan_xmit_frame
                                                      └─i40e_xmit_frame_ring
                                                           └─i40e_tx_map
                                                              ├─dma_map_single
                                                              └─netdev_tx_sent_queue

```

经过代码分析，发现ovs kernel datapath vxlan完全利用了kernel vxlan，那ovs dpdk datapath vxlan怎么实现的呢，vxlan encap ip lookup和arp怎么处理的呢，vpp的vxlan怎么实现的，相比ovs实现有什么区别，请听后面分解。

![](/assets/network-virtualnet-ovs-vxlan-dataflow1.png)

