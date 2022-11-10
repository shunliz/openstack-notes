# L2&L3

```
[root@Centrial ~]# ovs-ofctl dump-flows br-int


### 1、接收流程
#     接收tunnel口进入的跨主机流量，恢复寄存器，metadata==tunnelid==datapathid，reg14==in_port， reg15==out_port，
#     封装在[28][29]中设置
#     虚拟机跨主机的流量需要通过tunnel口发出，但在这之前，转发流程已经在源主机上确认了，即当前是在那个逻辑设备上（metadata）、从那个口收（reg14）、从那个口出（reg15）都已经确认，
#     从tunnel口发出时，这些信息都会设置到tunnel的option中（可扩展的元数据），到了对端主机后，再取出设置到对应的寄存器中。
###
 cookie=0x0, duration=101008.695s, table=0, n_packets=26, n_bytes=2548, priority=100,in_port="ovn-ba702e-0" actions=move:NXM_NX_TUN_ID[0..23]->OXM_OF_METADATA[0..23],move:NXM_NX_TUN_METADATA0[16..30]->NXM_NX_REG14[0..14],move:NXM_NX_TUN_METADATA0[0..15]->NXM_NX_REG15[0..15],resubmit(,33)


 ### 2、接收流程，接收vm进入的报文，设置寄存器，metadata=datapathid, reg14=input_if_id
 cookie=0x0, duration=96240.250s, table=0, n_packets=37, n_bytes=3218, priority=100,in_port="sw-300-port-vm1" actions=load:0x1->NXM_NX_REG13[],load:0x2->NXM_NX_REG11[],load:0x3->NXM_NX_REG12[],load:0x5->OXM_OF_METADATA[],load:0x1->NXM_NX_REG14[],resubmit(,8)
 cookie=0x0, duration=96239.640s, table=0, n_packets=35, n_bytes=3134, priority=100,in_port="sw-300-port-vm2" actions=load:0x4->NXM_NX_REG13[],load:0x2->NXM_NX_REG11[],load:0x3->NXM_NX_REG12[],load:0x5->OXM_OF_METADATA[],load:0x2->NXM_NX_REG14[],resubmit(,8)
 cookie=0x0, duration=87681.565s, table=0, n_packets=35, n_bytes=3246, priority=100,in_port="sw-400-port-vm1" actions=load:0x5->NXM_NX_REG13[],load:0x6->NXM_NX_REG11[],load:0x7->NXM_NX_REG12[],load:0x6->OXM_OF_METADATA[],load:0x1->NXM_NX_REG14[],resubmit(,8)

 ### 非法报文检测，对vlan和smac合法性做检查 1）不能带有vlan，很严格的检查； 2）smac不能是广播、组播mac
 cookie=0xce7f9f29, duration=96240.249s, table=8, n_packets=0, n_bytes=0, priority=100,metadata=0x5,vlan_tci=0x1000/0x1000 actions=drop
 cookie=0x572b2d0f, duration=87681.563s, table=8, n_packets=0, n_bytes=0, priority=100,metadata=0x6,vlan_tci=0x1000/0x1000 actions=drop
 cookie=0x1b96db9, duration=79156.683s, table=8, n_packets=0, n_bytes=0, priority=100,metadata=0x7,vlan_tci=0x1000/0x1000 actions=drop
 cookie=0xab874b4, duration=96240.248s, table=8, n_packets=0, n_bytes=0, priority=100,metadata=0x5,dl_src=01:00:00:00:00:00/01:00:00:00:00:00 actions=drop
 cookie=0x8a384251, duration=87681.563s, table=8, n_packets=0, n_bytes=0, priority=100,metadata=0x6,dl_src=01:00:00:00:00:00/01:00:00:00:00:00 actions=drop
 cookie=0x1b96db9, duration=79156.683s, table=8, n_packets=0, n_bytes=0, priority=100,metadata=0x7,dl_src=01:00:00:00:00:00/01:00:00:00:00:00 actions=drop


 cookie=0xfc6369dc, duration=88638.930s, table=8, n_packets=0, n_bytes=0, priority=50,reg14=0x1,metadata=0x5,dl_src=fa:10:dd:1b:30:01 actions=resubmit(,9)
 cookie=0x7985c8ec, duration=88621.461s, table=8, n_packets=0, n_bytes=0, priority=50,reg14=0x2,metadata=0x5,dl_src=fa:10:dd:1b:30:02 actions=resubmit(,9)
 cookie=0xd0e12cee, duration=87681.562s, table=8, n_packets=34, n_bytes=3156, priority=50,reg14=0x1,metadata=0x6,dl_src=fa:10:dd:1b:40:01 actions=resubmit(,9)
 cookie=0xd41d6822, duration=79156.688s, table=8, n_packets=0, n_bytes=0, priority=50,reg14=0x3,metadata=0x5 actions=resubmit(,9)
 cookie=0xfb376ec4, duration=79156.670s, table=8, n_packets=0, n_bytes=0, priority=50,reg14=0x3,metadata=0x6 actions=resubmit(,9)
 cookie=0x5f864969, duration=79156.684s, table=8, n_packets=0, n_bytes=0, priority=50,reg14=0x2,metadata=0x7,dl_dst=01:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,9)
 cookie=0xacd96dc9, duration=79156.683s, table=8, n_packets=0, n_bytes=0, priority=50,reg14=0x1,metadata=0x7,dl_dst=01:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,9)
 cookie=0xa0308908, duration=79156.683s, table=8, n_packets=0, n_bytes=0, priority=50,reg14=0x1,metadata=0x7,dl_dst=02:d4:1d:8c:30:01 actions=resubmit(,9)
 cookie=0x670f23f7, duration=79156.683s, table=8, n_packets=0, n_bytes=0, priority=50,reg14=0x2,metadata=0x7,dl_dst=02:d4:1d:8c:40:01 actions=resubmit(,9)

 cookie=0x2f8b8135, duration=79156.684s, table=9, n_packets=0, n_bytes=0, priority=100,ip,metadata=0x7,nw_dst=224.0.0.0/4 actions=drop
 cookie=0x80287555, duration=79156.684s, table=9, n_packets=0, n_bytes=0, priority=100,ip,reg9=0/0x2,metadata=0x7,nw_src=30.1.1.1 actions=drop
 cookie=0x8404f27d, duration=79156.683s, table=9, n_packets=0, n_bytes=0, priority=100,ip,reg9=0/0x2,metadata=0x7,nw_src=40.1.1.1 actions=drop
 cookie=0x80287555, duration=79156.683s, table=9, n_packets=0, n_bytes=0, priority=100,ip,reg9=0/0x2,metadata=0x7,nw_src=30.1.1.255 actions=drop
 cookie=0x8404f27d, duration=79156.682s, table=9, n_packets=0, n_bytes=0, priority=100,ip,reg9=0/0x2,metadata=0x7,nw_src=40.1.1.255 actions=drop
 cookie=0x2f8b8135, duration=79156.683s, table=9, n_packets=0, n_bytes=0, priority=100,ip,metadata=0x7,nw_dst=0.0.0.0/8 actions=drop
 cookie=0x2f8b8135, duration=79156.683s, table=9, n_packets=0, n_bytes=0, priority=100,ip,metadata=0x7,nw_dst=127.0.0.0/8 actions=drop
 cookie=0x2f8b8135, duration=79156.683s, table=9, n_packets=0, n_bytes=0, priority=100,ip,metadata=0x7,nw_src=0.0.0.0/8 actions=drop
 cookie=0x2f8b8135, duration=79156.683s, table=9, n_packets=0, n_bytes=0, priority=100,ip,metadata=0x7,nw_src=127.0.0.0/8 actions=drop
 cookie=0x2f8b8135, duration=79156.683s, table=9, n_packets=0, n_bytes=0, priority=100,ip,metadata=0x7,nw_src=255.255.255.255 actions=drop


 ### 3、port-security配置额外生成，接收的源检查，保证vm port和ip+mac的一一对应关系。不对应的，在下面[8]中 drop
 #      同时，目的vm的ip+mac的对应关系也会做检查，在 [36] 中
 ###
 cookie=0xb7abc8b4, duration=88638.930s, table=9, n_packets=0, n_bytes=0, priority=90,ip,reg14=0x1,metadata=0x5,dl_src=fa:10:dd:1b:30:01,nw_src=30.1.1.11 actions=resubmit(,10)
 cookie=0x4b18573, duration=88621.461s, table=9, n_packets=0, n_bytes=0, priority=90,ip,reg14=0x2,metadata=0x5,dl_src=fa:10:dd:1b:30:02,nw_src=30.1.1.12 actions=resubmit(,10)
 cookie=0x78a7ca35, duration=87681.562s, table=9, n_packets=26, n_bytes=2548, priority=90,ip,reg14=0x1,metadata=0x6,dl_src=fa:10:dd:1b:40:01,nw_src=40.1.1.11 actions=resubmit(,10)





 ### 4、放开dhcp报文
 cookie=0xde0e4e95, duration=88638.930s, table=9, n_packets=0, n_bytes=0, priority=90,udp,reg14=0x1,metadata=0x5,dl_src=fa:10:dd:1b:30:01,nw_src=0.0.0.0,nw_dst=255.255.255.255,tp_src=68,tp_dst=67 actions=resubmit(,10)
 cookie=0x5195c7cb, duration=88621.461s, table=9, n_packets=0, n_bytes=0, priority=90,udp,reg14=0x2,metadata=0x5,dl_src=fa:10:dd:1b:30:02,nw_src=0.0.0.0,nw_dst=255.255.255.255,tp_src=68,tp_dst=67 actions=resubmit(,10)
 cookie=0xc9447e1e, duration=87681.562s, table=9, n_packets=0, n_bytes=0, priority=90,udp,reg14=0x1,metadata=0x6,dl_src=fa:10:dd:1b:40:01,nw_src=0.0.0.0,nw_dst=255.255.255.255,tp_src=68,tp_dst=67 actions=resubmit(,10)


 ### 5、
 # mac地址学习功能：   === 针对的一些非托管的外部port，vm的port mac都是静态转发的
 # 字面意思 reg0=arp_spa, eth_src=arp_sha, 送控制器处理{put_arp(inport, arp.spa, arp.sha))}，恢复寄存器；userdata的前四个字节是opcode，这里是0x01，表示opcode是ACTION_OPCODE_PUT_ARP
 # 这条流表需要和 [21] 配合理解，logic router确定了nexthop的转发端口，用get_arp action来查找nexthop的mac是否已经学习到（在table 66里面）。
 # mac地址学习的流程是 20 --> 21 --> 5 --> 66
 # logic router确定了nexthop的转发端口，用get_arp action[20]来查找nexthop的mac是否已经学习到（在table 66里面），如果没有在table 66 修改mac成功，会在 [21]中
 # match dl_dst=00:00:00:00:00:00，控制器会从router转发端口广播一个arp request，logic switch会广播到它下的每一个端口，如果有arp应答，logic switch会回给logic router，
 # logic router中会通过下面这条流表，执行put_arp action，学到 arp。ovn-controller会在MAC_Binding表里增加一行记录。ovn-controller收到MAC_Binding表的更新后，添加一条flow到table=66里面；
 # 下次再走到 [20]的时候就能修改下一跳mac地址了。
 # 至此流程结束
 ###
 cookie=0x91fc8a10, duration=79156.684s, table=9, n_packets=0, n_bytes=0, priority=90,arp,metadata=0x7,arp_op=2 actions=push:NXM_NX_REG0[],push:NXM_OF_ETH_SRC[],push:NXM_NX_ARP_SHA[],push:NXM_OF_ARP_SPA[],pop:NXM_NX_REG0[],pop:NXM_OF_ETH_SRC[],controller(userdata=00.00.00.01.00.00.00.00),pop:NXM_OF_ETH_SRC[],pop:NXM_NX_REG0[]

 ### 6、网关地址的icmp代答，时刻注意，OVN的网关地址是逻辑的、不存在的，无法利用协议栈做任何事情，这里针对icmp做代答，模拟协议栈回icmp reply
 ###    包括下面的 [7]都是一个道理。
 cookie=0x623d5347, duration=79156.683s, table=9, n_packets=0, n_bytes=0, priority=90,icmp,metadata=0x7,nw_dst=30.1.1.1,icmp_type=8,icmp_code=0 actions=push:NXM_OF_IP_SRC[],push:NXM_OF_IP_DST[],pop:NXM_OF_IP_SRC[],pop:NXM_OF_IP_DST[],load:0xff->NXM_NX_IP_TTL[],load:0->NXM_OF_ICMP_TYPE[],load:0x1->NXM_NX_REG10[0],resubmit(,10)
 cookie=0xd1ba4b16, duration=79156.683s, table=9, n_packets=0, n_bytes=0, priority=90,icmp,metadata=0x7,nw_dst=40.1.1.1,icmp_type=8,icmp_code=0 actions=push:NXM_OF_IP_SRC[],push:NXM_OF_IP_DST[],pop:NXM_OF_IP_SRC[],pop:NXM_OF_IP_DST[],load:0xff->NXM_NX_IP_TTL[],load:0->NXM_OF_ICMP_TYPE[],load:0x1->NXM_NX_REG10[0],resubmit(,10)


 ### 7、router上网关地址的arp 代答，reg0=arp_spa, eth_src=arp_sha, reg15=2（出接口确认）, reg10=1(arp代答)标记，这里也做了 put_arp处理，控制器学习源arp
 cookie=0x732da042, duration=79156.683s, table=9, n_packets=0, n_bytes=0, priority=90,arp,reg14=0x2,metadata=0x7,arp_spa=40.1.1.0/24,arp_tpa=40.1.1.1,arp_op=1 actions=push:NXM_NX_REG0[],push:NXM_OF_ETH_SRC[],push:NXM_NX_ARP_SHA[],push:NXM_OF_ARP_SPA[],pop:NXM_NX_REG0[],pop:NXM_OF_ETH_SRC[],controller(userdata=00.00.00.01.00.00.00.00),pop:NXM_OF_ETH_SRC[],pop:NXM_NX_REG0[],move:NXM_OF_ETH_SRC[]->NXM_OF_ETH_DST[],mod_dl_src:02:d4:1d:8c:40:01,load:0x2->NXM_OF_ARP_OP[],move:NXM_NX_ARP_SHA[]->NXM_NX_ARP_THA[],load:0x2d41d8c4001->NXM_NX_ARP_SHA[],move:NXM_OF_ARP_SPA[]->NXM_OF_ARP_TPA[],load:0x28010101->NXM_OF_ARP_SPA[],load:0x2->NXM_NX_REG15[],load:0x1->NXM_NX_REG10[0],resubmit(,32)
 cookie=0x220cae3f, duration=79156.683s, table=9, n_packets=0, n_bytes=0, priority=90,arp,reg14=0x1,metadata=0x7,arp_spa=30.1.1.0/24,arp_tpa=30.1.1.1,arp_op=1 actions=push:NXM_NX_REG0[],push:NXM_OF_ETH_SRC[],push:NXM_NX_ARP_SHA[],push:NXM_OF_ARP_SPA[],pop:NXM_NX_REG0[],pop:NXM_OF_ETH_SRC[],controller(userdata=00.00.00.01.00.00.00.00),pop:NXM_OF_ETH_SRC[],pop:NXM_NX_REG0[],move:NXM_OF_ETH_SRC[]->NXM_OF_ETH_DST[],mod_dl_src:02:d4:1d:8c:30:01,load:0x2->NXM_OF_ARP_OP[],move:NXM_NX_ARP_SHA[]->NXM_NX_ARP_THA[],load:0x2d41d8c3001->NXM_NX_ARP_SHA[],move:NXM_OF_ARP_SPA[]->NXM_OF_ARP_TPA[],load:0x1e010101->NXM_OF_ARP_SPA[],load:0x1->NXM_NX_REG15[],load:0x1->NXM_NX_REG10[0],resubmit(,32)


 ### 8、ref [3]，mac+ip不对应的 drop
 cookie=0x3369cd66, duration=88638.930s, table=9, n_packets=0, n_bytes=0, priority=80,ip,reg14=0x1,metadata=0x5,dl_src=fa:10:dd:1b:30:01 actions=drop
 cookie=0xc93aa93d, duration=88621.461s, table=9, n_packets=0, n_bytes=0, priority=80,ip,reg14=0x2,metadata=0x5,dl_src=fa:10:dd:1b:30:02 actions=drop
 cookie=0x2852c9ee, duration=87681.563s, table=9, n_packets=0, n_bytes=0, priority=80,ip,reg14=0x1,metadata=0x6,dl_src=fa:10:dd:1b:40:01 actions=drop



 ### 9、其他来自 sw 的，非网关地址的arp request，上送控制器做 put_arp处理，控制器学习源arp。
 #     注意，1. 这是在router上的处理，2. 一般vm的arp是不会走到的，他们已经在sw上做了代答了，在[23]中。到这里的都是些未知的port发出的
 cookie=0xec3e3ecc, duration=79156.684s, table=9, n_packets=0, n_bytes=0, priority=80,arp,reg14=0x2,metadata=0x7,arp_spa=40.1.1.0/24,arp_op=1 actions=push:NXM_NX_REG0[],push:NXM_OF_ETH_SRC[],push:NXM_NX_ARP_SHA[],push:NXM_OF_ARP_SPA[],pop:NXM_NX_REG0[],pop:NXM_OF_ETH_SRC[],controller(userdata=00.00.00.01.00.00.00.00),pop:NXM_OF_ETH_SRC[],pop:NXM_NX_REG0[]
 cookie=0xe8db4d6e, duration=79156.683s, table=9, n_packets=0, n_bytes=0, priority=80,arp,reg14=0x1,metadata=0x7,arp_spa=30.1.1.0/24,arp_op=1 actions=push:NXM_NX_REG0[],push:NXM_OF_ETH_SRC[],push:NXM_NX_ARP_SHA[],push:NXM_OF_ARP_SPA[],pop:NXM_NX_REG0[],pop:NXM_OF_ETH_SRC[],controller(userdata=00.00.00.01.00.00.00.00),pop:NXM_OF_ETH_SRC[],pop:NXM_NX_REG0[]


### ����������tcp�� �tcp_reset
### ��������udp����icmp Port Unreachable������� ��
### ����������ip���� Protocol Unreachable�������

### 10、如下是对到router网关的报文的处理，同[6][7]一个道理，网关地址是不存在的，为了更好的模拟协议栈，对tcp、udp、ip分别做了回复处理。
# tcp，构造tcp_reset 报文回复
# udp，构造icmp Port Unreachable������� 回复
# 其他ip，构造 icmp Protocol Unreachable���� ���回复
# 未匹配的全部drop，也就是说目的地址是网关的，到这里为止，出了icmp 和 arp request，全部丢弃
###
 cookie=0x1bf2c714, duration=79156.684s, table=9, n_packets=0, n_bytes=0, priority=80,tcp,metadata=0x7,nw_dst=30.1.1.1,nw_frag=not_later actions=controller(userdata=00.00.00.0b.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1b.00.00.00.00.04.06.00.30.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1b.00.00.00.00.02.06.00.30.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1c.00.00.00.00.04.06.00.30.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1c.00.00.00.00.02.06.00.30.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1b.00.00.00.00.0e.04.00.20.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1b.00.00.00.00.10.04.00.20.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1c.00.00.00.00.0e.04.00.20.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1c.00.00.00.00.10.04.00.20.00.00.00.00.00.00.ff.ff.00.10.00.00.23.20.00.0e.ff.f8.0a.00.00.00)
 cookie=0x7703c2e1, duration=79156.683s, table=9, n_packets=0, n_bytes=0, priority=80,udp,metadata=0x7,nw_dst=30.1.1.1,nw_frag=not_later actions=controller(userdata=00.00.00.0a.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1b.00.00.00.00.04.06.00.30.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1b.00.00.00.00.02.06.00.30.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1c.00.00.00.00.04.06.00.30.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1c.00.00.00.00.02.06.00.30.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1b.00.00.00.00.0e.04.00.20.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1b.00.00.00.00.10.04.00.20.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1c.00.00.00.00.0e.04.00.20.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1c.00.00.00.00.10.04.00.20.00.00.00.00.00.00.00.19.00.10.00.01.3a.01.ff.00.00.00.00.00.00.00.00.19.00.10.80.00.26.01.03.00.00.00.00.00.00.00.00.19.00.10.80.00.28.01.03.00.00.00.00.00.00.00.ff.ff.00.10.00.00.23.20.00.0e.ff.f8.0a.00.00.00)
 cookie=0x881f5578, duration=79156.683s, table=9, n_packets=0, n_bytes=0, priority=80,udp,metadata=0x7,nw_dst=40.1.1.1,nw_frag=not_later actions=controller(userdata=00.00.00.0a.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1b.00.00.00.00.04.06.00.30.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1b.00.00.00.00.02.06.00.30.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1c.00.00.00.00.04.06.00.30.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1c.00.00.00.00.02.06.00.30.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1b.00.00.00.00.0e.04.00.20.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1b.00.00.00.00.10.04.00.20.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1c.00.00.00.00.0e.04.00.20.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1c.00.00.00.00.10.04.00.20.00.00.00.00.00.00.00.19.00.10.00.01.3a.01.ff.00.00.00.00.00.00.00.00.19.00.10.80.00.26.01.03.00.00.00.00.00.00.00.00.19.00.10.80.00.28.01.03.00.00.00.00.00.00.00.ff.ff.00.10.00.00.23.20.00.0e.ff.f8.0a.00.00.00)
 cookie=0x5bcf8f33, duration=79156.683s, table=9, n_packets=0, n_bytes=0, priority=80,tcp,metadata=0x7,nw_dst=40.1.1.1,nw_frag=not_later actions=controller(userdata=00.00.00.0b.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1b.00.00.00.00.04.06.00.30.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1b.00.00.00.00.02.06.00.30.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1c.00.00.00.00.04.06.00.30.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1c.00.00.00.00.02.06.00.30.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1b.00.00.00.00.0e.04.00.20.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1b.00.00.00.00.10.04.00.20.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1c.00.00.00.00.0e.04.00.20.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1c.00.00.00.00.10.04.00.20.00.00.00.00.00.00.ff.ff.00.10.00.00.23.20.00.0e.ff.f8.0a.00.00.00)
 cookie=0xf7515999, duration=79156.684s, table=9, n_packets=0, n_bytes=0, priority=70,ip,metadata=0x7,nw_dst=30.1.1.1,nw_frag=not_later actions=controller(userdata=00.00.00.0a.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1b.00.00.00.00.04.06.00.30.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1b.00.00.00.00.02.06.00.30.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1c.00.00.00.00.04.06.00.30.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1c.00.00.00.00.02.06.00.30.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1b.00.00.00.00.0e.04.00.20.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1b.00.00.00.00.10.04.00.20.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1c.00.00.00.00.0e.04.00.20.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1c.00.00.00.00.10.04.00.20.00.00.00.00.00.00.00.19.00.10.00.01.3a.01.ff.00.00.00.00.00.00.00.00.19.00.10.80.00.26.01.03.00.00.00.00.00.00.00.00.19.00.10.80.00.28.01.02.00.00.00.00.00.00.00.ff.ff.00.10.00.00.23.20.00.0e.ff.f8.0a.00.00.00)
 cookie=0xf232f747, duration=79156.683s, table=9, n_packets=0, n_bytes=0, priority=70,ip,metadata=0x7,nw_dst=40.1.1.1,nw_frag=not_later actions=controller(userdata=00.00.00.0a.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1b.00.00.00.00.04.06.00.30.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1b.00.00.00.00.02.06.00.30.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1c.00.00.00.00.04.06.00.30.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1c.00.00.00.00.02.06.00.30.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1b.00.00.00.00.0e.04.00.20.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1b.00.00.00.00.10.04.00.20.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1c.00.00.00.00.0e.04.00.20.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1c.00.00.00.00.10.04.00.20.00.00.00.00.00.00.00.19.00.10.00.01.3a.01.ff.00.00.00.00.00.00.00.00.19.00.10.80.00.26.01.03.00.00.00.00.00.00.00.00.19.00.10.80.00.28.01.02.00.00.00.00.00.00.00.ff.ff.00.10.00.00.23.20.00.0e.ff.f8.0a.00.00.00)
 cookie=0x6982c737, duration=79156.683s, table=9, n_packets=0, n_bytes=0, priority=60,ip,metadata=0x7,nw_dst=30.1.1.1 actions=drop
 cookie=0x760a445f, duration=79156.683s, table=9, n_packets=0, n_bytes=0, priority=60,ip,metadata=0x7,nw_dst=40.1.1.1 actions=drop


### 11、router上的其他广播，丢弃。
 cookie=0x72182bb2, duration=79156.683s, table=9, n_packets=0, n_bytes=0, priority=50,metadata=0x7,dl_dst=ff:ff:ff:ff:ff:ff actions=drop

### 12、这里处理ttl={0,1}的报文，即不可达报文, 如果是非分片或分片首包，则由控制器构造 "icmp TTL equals 0 during transit�����������"报文回复，否则drop
###     OVN做的真是细致。
 cookie=0xdbd78f03, duration=79156.683s, table=9, n_packets=0, n_bytes=0, priority=40,ip,reg14=0x1,metadata=0x7,nw_ttl=1,nw_frag=not_later actions=controller(userdata=00.00.00.0a.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1b.00.00.00.00.04.06.00.30.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1b.00.00.00.00.02.06.00.30.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1c.00.00.00.00.04.06.00.30.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1c.00.00.00.00.02.06.00.30.00.00.00.00.00.00.00.19.00.10.80.00.26.01.0b.00.00.00.00.00.00.00.00.19.00.10.80.00.28.01.00.00.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.06.00.20.00.00.00.00.00.00.0e.04.00.00.10.04.00.19.00.10.80.00.16.04.1e.01.01.01.00.00.00.00.00.19.00.10.00.01.3a.01.ff.00.00.00.00.00.00.00.ff.ff.00.10.00.00.23.20.00.0e.ff.f8.0a.00.00.00)
 cookie=0x6a19bf6a, duration=79156.683s, table=9, n_packets=0, n_bytes=0, priority=40,ip,reg14=0x2,metadata=0x7,nw_ttl=0,nw_frag=not_later actions=controller(userdata=00.00.00.0a.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1b.00.00.00.00.04.06.00.30.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1b.00.00.00.00.02.06.00.30.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1c.00.00.00.00.04.06.00.30.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1c.00.00.00.00.02.06.00.30.00.00.00.00.00.00.00.19.00.10.80.00.26.01.0b.00.00.00.00.00.00.00.00.19.00.10.80.00.28.01.00.00.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.06.00.20.00.00.00.00.00.00.0e.04.00.00.10.04.00.19.00.10.80.00.16.04.28.01.01.01.00.00.00.00.00.19.00.10.00.01.3a.01.ff.00.00.00.00.00.00.00.ff.ff.00.10.00.00.23.20.00.0e.ff.f8.0a.00.00.00)
 cookie=0xdbd78f03, duration=79156.683s, table=9, n_packets=0, n_bytes=0, priority=40,ip,reg14=0x1,metadata=0x7,nw_ttl=0,nw_frag=not_later actions=controller(userdata=00.00.00.0a.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1b.00.00.00.00.04.06.00.30.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1b.00.00.00.00.02.06.00.30.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1c.00.00.00.00.04.06.00.30.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1c.00.00.00.00.02.06.00.30.00.00.00.00.00.00.00.19.00.10.80.00.26.01.0b.00.00.00.00.00.00.00.00.19.00.10.80.00.28.01.00.00.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.06.00.20.00.00.00.00.00.00.0e.04.00.00.10.04.00.19.00.10.80.00.16.04.1e.01.01.01.00.00.00.00.00.19.00.10.00.01.3a.01.ff.00.00.00.00.00.00.00.ff.ff.00.10.00.00.23.20.00.0e.ff.f8.0a.00.00.00)
 cookie=0x6a19bf6a, duration=79156.683s, table=9, n_packets=0, n_bytes=0, priority=40,ip,reg14=0x2,metadata=0x7,nw_ttl=1,nw_frag=not_later actions=controller(userdata=00.00.00.0a.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1b.00.00.00.00.04.06.00.30.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1b.00.00.00.00.02.06.00.30.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1c.00.00.00.00.04.06.00.30.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.1c.00.00.00.00.02.06.00.30.00.00.00.00.00.00.00.19.00.10.80.00.26.01.0b.00.00.00.00.00.00.00.00.19.00.10.80.00.28.01.00.00.00.00.00.00.00.00.ff.ff.00.18.00.00.23.20.00.06.00.20.00.00.00.00.00.00.0e.04.00.00.10.04.00.19.00.10.80.00.16.04.28.01.01.01.00.00.00.00.00.19.00.10.00.01.3a.01.ff.00.00.00.00.00.00.00.ff.ff.00.10.00.00.23.20.00.0e.ff.f8.0a.00.00.00)
 cookie=0x57b1321d, duration=79156.684s, table=9, n_packets=0, n_bytes=0, priority=30,ip,metadata=0x7,nw_ttl=1 actions=drop
 cookie=0x57b1321d, duration=79156.683s, table=9, n_packets=0, n_bytes=0, priority=30,ip,metadata=0x7,nw_ttl=0 actions=drop



 cookie=0x74b39f35, duration=96240.249s, table=9, n_packets=72, n_bytes=6352, priority=0,metadata=0x5 actions=resubmit(,10)
 cookie=0x7c4115fa, duration=87681.563s, table=9, n_packets=1, n_bytes=42, priority=0,metadata=0x6 actions=resubmit(,10)
 cookie=0x3846a8f1, duration=79156.683s, table=9, n_packets=0, n_bytes=0, priority=0,metadata=0x7 actions=resubmit(,10)

### 13、logic sw中 vm发出的arp报文，next，这部分不包含请求网关的（其在[7]中已经代答），在下面 [23]中代答；其他未知的arp，drop
 cookie=0xb8a417ee, duration=88638.930s, table=10, n_packets=0, n_bytes=0, priority=90,arp,reg14=0x1,metadata=0x5,dl_src=fa:10:dd:1b:30:01,arp_spa=30.1.1.11,arp_sha=fa:10:dd:1b:30:01 actions=resubmit(,11)
 cookie=0x168bcf46, duration=88621.461s, table=10, n_packets=0, n_bytes=0, priority=90,arp,reg14=0x2,metadata=0x5,dl_src=fa:10:dd:1b:30:02,arp_spa=30.1.1.12,arp_sha=fa:10:dd:1b:30:02 actions=resubmit(,11)
 cookie=0x4611c4a2, duration=87681.563s, table=10, n_packets=1, n_bytes=42, priority=90,arp,reg14=0x1,metadata=0x6,dl_src=fa:10:dd:1b:40:01,arp_spa=40.1.1.11,arp_sha=fa:10:dd:1b:40:01 actions=resubmit(,11)
 cookie=0x334b415e, duration=88638.930s, table=10, n_packets=0, n_bytes=0, priority=80,arp,reg14=0x1,metadata=0x5 actions=drop
 cookie=0x3cdad3cd, duration=88621.461s, table=10, n_packets=0, n_bytes=0, priority=80,arp,reg14=0x2,metadata=0x5 actions=drop
 cookie=0x39a221b2, duration=87681.563s, table=10, n_packets=0, n_bytes=0, priority=80,arp,reg14=0x1,metadata=0x6 actions=drop


### next�next......
 cookie=0xd72dc9ec, duration=96240.249s, table=10, n_packets=72, n_bytes=6352, priority=0,metadata=0x5 actions=resubmit(,11)
 cookie=0x914db8cb, duration=87681.563s, table=10, n_packets=26, n_bytes=2548, priority=0,metadata=0x6 actions=resubmit(,11)
 cookie=0x83a968d6, duration=79156.684s, table=10, n_packets=0, n_bytes=0, priority=0,metadata=0x7 actions=resubmit(,11)
 cookie=0x6637dd4d, duration=96240.249s, table=11, n_packets=72, n_bytes=6352, priority=0,metadata=0x5 actions=resubmit(,12)
 cookie=0xaa003dd3, duration=87681.563s, table=11, n_packets=27, n_bytes=2590, priority=0,metadata=0x6 actions=resubmit(,12)
 cookie=0x3bdef87, duration=79156.684s, table=11, n_packets=0, n_bytes=0, priority=0,metadata=0x7 actions=resubmit(,12)
 cookie=0x7d10478a, duration=96240.249s, table=12, n_packets=64, n_bytes=5760, priority=0,metadata=0x5 actions=resubmit(,13)
 cookie=0x662de14f, duration=87681.563s, table=12, n_packets=27, n_bytes=2590, priority=0,metadata=0x6 actions=resubmit(,13)
 cookie=0x2f100e98, duration=79156.683s, table=12, n_packets=0, n_bytes=0, priority=0,metadata=0x7 actions=resubmit(,13)


#### 14、ct ��������相关，nat、安全组等功能，本节未做配置
 cookie=0xb67c82a9, duration=96240.248s, table=13, n_packets=0, n_bytes=0, priority=100,ip,reg0=0x1/0x1,metadata=0x5 actions=ct(table=14,zone=NXM_NX_REG13[0..15])
 cookie=0x65cf352e, duration=87681.563s, table=13, n_packets=0, n_bytes=0, priority=100,ip,reg0=0x1/0x1,metadata=0x6 actions=ct(table=14,zone=NXM_NX_REG13[0..15])


 cookie=0xc621f835, duration=96240.248s, table=13, n_packets=72, n_bytes=6352, priority=0,metadata=0x5 actions=resubmit(,14)
 cookie=0x1c44a08e, duration=87681.563s, table=13, n_packets=27, n_bytes=2590, priority=0,metadata=0x6 actions=resubmit(,14)
 cookie=0x73bbe239, duration=79156.683s, table=13, n_packets=0, n_bytes=0, priority=0,metadata=0x7 actions=resubmit(,14)
 cookie=0x20417229, duration=96240.249s, table=14, n_packets=72, n_bytes=6352, priority=0,metadata=0x5 actions=resubmit(,15)
 cookie=0x93196d05, duration=87681.563s, table=14, n_packets=27, n_bytes=2590, priority=0,metadata=0x6 actions=resubmit(,15)
 cookie=0x416e25d5, duration=79156.683s, table=14, n_packets=0, n_bytes=0, priority=0,metadata=0x7 actions=resubmit(,15)

### 15、逻辑路由器上转发到vm的报文，为转发修改报文，同内核协议栈一样，这里先修改ttl 和 smac {ttl--, eth.src = 02:d4:1d:8c:40:01}�，在[16]中修改dmac
#       reg0=XXREG0[96..127]=ip4.dst, reg1=XXREG0[64..95]={40.1.1.1, }, reg15=2(output:rt-400-port，确认了转发出接口), reg10=1（标记着什么？？）
###
 cookie=0xcf105cd3, duration=79156.684s, table=15, n_packets=0, n_bytes=0, priority=49,ip,metadata=0x7,nw_dst=40.1.1.0/24 actions=dec_ttl(),move:NXM_OF_IP_DST[]->NXM_NX_XXREG0[96..127],load:0x28010101->NXM_NX_XXREG0[64..95],mod_dl_src:02:d4:1d:8c:40:01,load:0x2->NXM_NX_REG15[],load:0x1->NXM_NX_REG10[0],resubmit(,16)
 cookie=0x448ecdcd, duration=79156.683s, table=15, n_packets=0, n_bytes=0, priority=49,ip,metadata=0x7,nw_dst=30.1.1.0/24 actions=dec_ttl(),move:NXM_OF_IP_DST[]->NXM_NX_XXREG0[96..127],load:0x1e010101->NXM_NX_XXREG0[64..95],mod_dl_src:02:d4:1d:8c:30:01,load:0x1->NXM_NX_REG15[],load:0x1->NXM_NX_REG10[0],resubmit(,16)


 cookie=0x4e4c046f, duration=96240.249s, table=15, n_packets=72, n_bytes=6352, priority=0,metadata=0x5 actions=resubmit(,16)
 cookie=0x5573d398, duration=87681.563s, table=15, n_packets=27, n_bytes=2590, priority=0,metadata=0x6 actions=resubmit(,16)

 ### 16、router转发到vm的报文，[15] 中修改了ttl 和 smac，这里修改dmac，switch中的流量next
 cookie=0x2787315d, duration=79156.684s, table=16, n_packets=0, n_bytes=0, priority=100,reg0=0x1e01010b,reg15=0x1,metadata=0x7 actions=mod_dl_dst:fa:10:dd:1b:30:01,resubmit(,17)
 cookie=0x470cf565, duration=79156.683s, table=16, n_packets=0, n_bytes=0, priority=100,reg0=0x1e01010c,reg15=0x1,metadata=0x7 actions=mod_dl_dst:fa:10:dd:1b:30:02,resubmit(,17)
 cookie=0x1ba0dfc4, duration=195.495s, table=16, n_packets=2, n_bytes=196, priority=100,reg0=0x2801010b,reg15=0x2,metadata=0x7 actions=mod_dl_dst:fa:10:dd:1b:40:01,resubmit(,17)
 cookie=0x836944c, duration=195.495s, table=16, n_packets=0, n_bytes=0, priority=100,reg0=0x2801010c,reg15=0x2,metadata=0x7 actions=mod_dl_dst:fa:10:dd:1b:40:02,resubmit(,17)
 ### 17、交换机中的则不需要做报文修改
 cookie=0xaba0fff5, duration=96240.249s, table=16, n_packets=72, n_bytes=6352, priority=0,metadata=0x5 actions=resubmit(,17)
 cookie=0xd09ca65e, duration=87681.563s, table=16, n_packets=27, n_bytes=2590, priority=0,metadata=0x6 actions=resubmit(,17)


 ### =========== 这里转发完成，确认了出接口，修改了ttl、smac、dmac，下面涉及转发后的操作


### 20、ref [5]，router上的ip报文，设置reg0=ip4.dst(更广泛的来说是下一跳地址，直连的才是dst，但直连的在上面 [16] [17] 中已经转到 table17了，不会走到这里), 
#       设置 dmac=00:00:00:00:00:00, 然后去table66修改dmac，恢复reg0，next到table17，走到这里的都不会是到vm的流量，一般是下一跳不在系统管理的，如去公网的流量
###
 cookie=0xd5478283, duration=79156.683s, table=16, n_packets=0, n_bytes=0, priority=0,ip,metadata=0x7 actions=push:NXM_NX_REG0[],push:NXM_NX_XXREG0[96..127],pop:NXM_NX_REG0[],mod_dl_dst:00:00:00:00:00:00,resubmit(,66),pop:NXM_NX_REG0[],resubmit(,17)


 cookie=0xc81907db, duration=96240.249s, table=17, n_packets=72, n_bytes=6352, priority=0,metadata=0x5 actions=resubmit(,18)
 cookie=0xa472f92f, duration=87681.562s, table=17, n_packets=27, n_bytes=2590, priority=0,metadata=0x6 actions=resubmit(,18)
 cookie=0x2f7d5364, duration=79156.683s, table=17, n_packets=0, n_bytes=0, priority=0,metadata=0x7 actions=resubmit(,18)
 cookie=0xe1bbed5a, duration=96240.249s, table=18, n_packets=0, n_bytes=0, priority=100,ip,reg0=0x2/0x2,metadata=0x5 actions=ct(commit,zone=NXM_NX_REG13[0..15],exec(load:0->NXM_NX_CT_LABEL[0])),resubmit(,19)
 cookie=0xb6359085, duration=87681.562s, table=18, n_packets=0, n_bytes=0, priority=100,ip,reg0=0x2/0x2,metadata=0x6 actions=ct(commit,zone=NXM_NX_REG13[0..15],exec(load:0->NXM_NX_CT_LABEL[0])),resubmit(,19)
 cookie=0xf049421, duration=96240.249s, table=18, n_packets=0, n_bytes=0, priority=100,ip,reg0=0x4/0x4,metadata=0x5 actions=ct(table=19,zone=NXM_NX_REG13[0..15],nat)
 cookie=0x80c13bc3, duration=87681.563s, table=18, n_packets=0, n_bytes=0, priority=100,ip,reg0=0x4/0x4,metadata=0x6 actions=ct(table=19,zone=NXM_NX_REG13[0..15],nat)


 ### 21、ref [5]，接[20]的处理，如果没有在table 66 修改mac成功，match dl_dst=00:00:00:00:00:00，控制器会从router转发端口广播一个arp request，
 # logic switch会广播到它下的每一个端口，如果有arp应答，logic switch会回给logic router，logic router中会通过上面 [5] 中table=9中的的流表，执行put_arp action，学到 arp
 # ovn-controller会在MAC_Binding表里增加一行记录。ovn-controller收到MAC_Binding表的更新后，添加一条flow到table=66里面。至此流程结束
 ###
 cookie=0xb22051f5, duration=79156.683s, table=18, n_packets=0, n_bytes=0, priority=100,ip,metadata=0x7,dl_dst=00:00:00:00:00:00 actions=controller(userdata=00.00.00.00.00.00.00.00.00.19.00.10.80.00.06.06.ff.ff.ff.ff.ff.ff.00.00.ff.ff.00.18.00.00.23.20.00.06.00.20.00.40.00.00.00.01.de.10.00.00.20.04.ff.ff.00.18.00.00.23.20.00.06.00.20.00.60.00.00.00.01.de.10.00.00.22.04.00.19.00.10.80.00.2a.02.00.01.00.00.00.00.00.00.ff.ff.00.10.00.00.23.20.00.0e.ff.f8.20.00.00.00)



 cookie=0x85b83e9e, duration=96240.248s, table=18, n_packets=72, n_bytes=6352, priority=0,metadata=0x5 actions=resubmit(,19)
 cookie=0x3aea55a, duration=87681.563s, table=18, n_packets=27, n_bytes=2590, priority=0,metadata=0x6 actions=resubmit(,19)
 cookie=0x91155207, duration=79156.683s, table=18, n_packets=0, n_bytes=0, priority=0,metadata=0x7 actions=resubmit(,32)

 ### 22、vm的免费arp，自己请求自己
 cookie=0x72936dc6, duration=96240.249s, table=19, n_packets=0, n_bytes=0, priority=100,arp,reg14=0x1,metadata=0x5,arp_tpa=30.1.1.11,arp_op=1 actions=resubmit(,20)
 cookie=0xaefc1cee, duration=96239.640s, table=19, n_packets=0, n_bytes=0, priority=100,arp,reg14=0x2,metadata=0x5,arp_tpa=30.1.1.12,arp_op=1 actions=resubmit(,20)
 cookie=0xe68a53e9, duration=87681.563s, table=19, n_packets=0, n_bytes=0, priority=100,arp,reg14=0x1,metadata=0x6,arp_tpa=40.1.1.11,arp_op=1 actions=resubmit(,20)

 ### 23、vm的arp代答
 cookie=0xeeeb1de0, duration=96240.248s, table=19, n_packets=3, n_bytes=126, priority=50,arp,metadata=0x5,arp_tpa=30.1.1.11,arp_op=1 actions=move:NXM_OF_ETH_SRC[]->NXM_OF_ETH_DST[],mod_dl_src:fa:10:dd:1b:30:01,load:0x2->NXM_OF_ARP_OP[],move:NXM_NX_ARP_SHA[]->NXM_NX_ARP_THA[],load:0xfa10dd1b3001->NXM_NX_ARP_SHA[],move:NXM_OF_ARP_SPA[]->NXM_OF_ARP_TPA[],load:0x1e01010b->NXM_OF_ARP_SPA[],move:NXM_NX_REG14[]->NXM_NX_REG15[],load:0x1->NXM_NX_REG10[0],resubmit(,32)
 cookie=0x245a45e8, duration=96239.640s, table=19, n_packets=5, n_bytes=210, priority=50,arp,metadata=0x5,arp_tpa=30.1.1.12,arp_op=1 actions=move:NXM_OF_ETH_SRC[]->NXM_OF_ETH_DST[],mod_dl_src:fa:10:dd:1b:30:02,load:0x2->NXM_OF_ARP_OP[],move:NXM_NX_ARP_SHA[]->NXM_NX_ARP_THA[],load:0xfa10dd1b3002->NXM_NX_ARP_SHA[],move:NXM_OF_ARP_SPA[]->NXM_OF_ARP_TPA[],load:0x1e01010c->NXM_OF_ARP_SPA[],move:NXM_NX_REG14[]->NXM_NX_REG15[],load:0x1->NXM_NX_REG10[0],resubmit(,32)
 cookie=0xdd75a2a3, duration=87681.562s, table=19, n_packets=0, n_bytes=0, priority=50,arp,metadata=0x6,arp_tpa=40.1.1.11,arp_op=1 actions=move:NXM_OF_ETH_SRC[]->NXM_OF_ETH_DST[],mod_dl_src:fa:10:dd:1b:40:01,load:0x2->NXM_OF_ARP_OP[],move:NXM_NX_ARP_SHA[]->NXM_NX_ARP_THA[],load:0xfa10dd1b4001->NXM_NX_ARP_SHA[],move:NXM_OF_ARP_SPA[]->NXM_OF_ARP_TPA[],load:0x2801010b->NXM_OF_ARP_SPA[],move:NXM_NX_REG14[]->NXM_NX_REG15[],load:0x1->NXM_NX_REG10[0],resubmit(,32)
 cookie=0x590e4150, duration=87669.756s, table=19, n_packets=1, n_bytes=42, priority=50,arp,metadata=0x6,arp_tpa=40.1.1.12,arp_op=1 actions=move:NXM_OF_ETH_SRC[]->NXM_OF_ETH_DST[],mod_dl_src:fa:10:dd:1b:40:02,load:0x2->NXM_OF_ARP_OP[],move:NXM_NX_ARP_SHA[]->NXM_NX_ARP_THA[],load:0xfa10dd1b4002->NXM_NX_ARP_SHA[],move:NXM_OF_ARP_SPA[]->NXM_OF_ARP_TPA[],load:0x2801010c->NXM_OF_ARP_SPA[],move:NXM_NX_REG14[]->NXM_NX_REG15[],load:0x1->NXM_NX_REG10[0],resubmit(,32)


 cookie=0x777b5126, duration=96240.249s, table=19, n_packets=64, n_bytes=6016, priority=0,metadata=0x5 actions=resubmit(,20)
 cookie=0xa95d6fb7, duration=87681.563s, table=19, n_packets=26, n_bytes=2548, priority=0,metadata=0x6 actions=resubmit(,20)
 cookie=0x48696989, duration=96240.249s, table=20, n_packets=64, n_bytes=6016, priority=0,metadata=0x5 actions=resubmit(,21)
 cookie=0x5f224fab, duration=87681.562s, table=20, n_packets=26, n_bytes=2548, priority=0,metadata=0x6 actions=resubmit(,21)
 cookie=0xc412ee7, duration=96240.248s, table=21, n_packets=64, n_bytes=6016, priority=0,metadata=0x5 actions=resubmit(,22)
 cookie=0xf05391c6, duration=87681.563s, table=21, n_packets=26, n_bytes=2548, priority=0,metadata=0x6 actions=resubmit(,22)
 cookie=0xb49e92fc, duration=96240.249s, table=22, n_packets=64, n_bytes=6016, priority=0,metadata=0x5 actions=resubmit(,23)
 cookie=0x3746f497, duration=87681.562s, table=22, n_packets=26, n_bytes=2548, priority=0,metadata=0x6 actions=resubmit(,23)
 cookie=0x7d948f61, duration=96240.249s, table=23, n_packets=64, n_bytes=6016, priority=0,metadata=0x5 actions=resubmit(,24)
 cookie=0xe1515175, duration=87681.563s, table=23, n_packets=26, n_bytes=2548, priority=0,metadata=0x6 actions=resubmit(,24)

 ### 24、logic sw中的 组播、广播mac地址，设置reg15=0xffff（所有接口），走table32发送
 cookie=0xb077ce92, duration=96240.249s, table=24, n_packets=16, n_bytes=1312, priority=100,metadata=0x5,dl_dst=01:00:00:00:00:00/01:00:00:00:00:00 actions=load:0xffff->NXM_NX_REG15[],resubmit(,32)
 cookie=0x3aee71af, duration=87681.562s, table=24, n_packets=0, n_bytes=0, priority=100,metadata=0x6,dl_dst=01:00:00:00:00:00/01:00:00:00:00:00 actions=load:0xffff->NXM_NX_REG15[],resubmit(,32)

 ### 25、logic sw中，去往具体的vm mac的报文，设置出接口（reg15），走table32发送
 cookie=0x8cfc69ef, duration=96240.249s, table=24, n_packets=24, n_bytes=2352, priority=50,metadata=0x5,dl_dst=fa:10:dd:1b:30:01 actions=load:0x1->NXM_NX_REG15[],resubmit(,32)
 cookie=0x1326bb75, duration=96240.249s, table=24, n_packets=24, n_bytes=2352, priority=50,metadata=0x5,dl_dst=fa:10:dd:1b:30:02 actions=load:0x2->NXM_NX_REG15[],resubmit(,32)
 cookie=0xa9d9de34, duration=87681.563s, table=24, n_packets=26, n_bytes=2548, priority=50,metadata=0x6,dl_dst=fa:10:dd:1b:40:02 actions=load:0x2->NXM_NX_REG15[],resubmit(,32)
 cookie=0xbac6e0e1, duration=87681.563s, table=24, n_packets=0, n_bytes=0, priority=50,metadata=0x6,dl_dst=fa:10:dd:1b:40:01 actions=load:0x1->NXM_NX_REG15[],resubmit(,32)
 ### 26、logic sw中，去往网关 mac的报文，设置出接口（reg15），走table32发送
 cookie=0x1628cdab, duration=79156.688s, table=24, n_packets=0, n_bytes=0, priority=50,metadata=0x5,dl_dst=02:d4:1d:8c:30:01 actions=load:0x3->NXM_NX_REG15[],resubmit(,32)
 cookie=0xc613bcbf, duration=79156.670s, table=24, n_packets=0, n_bytes=0, priority=50,metadata=0x6,dl_dst=02:d4:1d:8c:40:01 actions=load:0x3->NXM_NX_REG15[],resubmit(,32)


 ### 27、reg10的这两个标记暂时没看到，可能其他功能标记，后续补充
 cookie=0x0, duration=279153.636s, table=32, n_packets=0, n_bytes=0, priority=150,reg10=0x10/0x10 actions=resubmit(,33)
 cookie=0x0, duration=279153.636s, table=32, n_packets=0, n_bytes=0, priority=150,reg10=0x2/0x2 actions=resubmit(,33)

 ### 28、ref[1] 跨主机流量，sw-400上的0x2的接口在另一台主机上，这里将sw_400的Datapath key作为tunnel id，
 #  出接口(sw-400-port-vm2)作为tun_metadata0（NXM_NX_TUN_METADATA0[0..15]），
 #  入接口作为 NXM_NX_TUN_METADATA0[16..30] 封装到tunnel头中，再走tunnel发送
 ###
 cookie=0x0, duration=87669.758s, table=32, n_packets=26, n_bytes=2548, priority=100,reg15=0x2,metadata=0x6 actions=load:0x6->NXM_NX_TUN_ID[0..23],set_field:0x2->tun_metadata0,move:NXM_NX_REG14[0..14]->NXM_NX_TUN_METADATA0[16..30],output:"ovn-ba702e-0"

 ### 29、跨主机流量，广播+组播，是要clone发送所有port的，但需要先经过table34检查是否是合法同接口进出，合法的，就会再走 [31][33] 在被检查的port上发送一份。
 #   这里是向路由口发一份，然后next 到 table33，在table33中会再次往vm接口中发送；
 #   其中sw-400中由于在其他主机也有vm所有会发一份到跨主机tunnel中
 ###
 cookie=0x0, duration=79156.688s, table=32, n_packets=0, n_bytes=0, priority=100,reg15=0xffff,metadata=0x5 actions=load:0x3->NXM_NX_REG15[],resubmit(,34),load:0xffff->NXM_NX_REG15[],resubmit(,33)
 cookie=0x0, duration=87669.758s, table=32, n_packets=0, n_bytes=0, priority=100,reg15=0xffff,metadata=0x6 actions=load:0x3->NXM_NX_REG15[],resubmit(,34),load:0xffff->NXM_NX_REG15[],load:0x6->NXM_NX_TUN_ID[0..23],set_field:0xffff->tun_metadata0,move:NXM_NX_REG14[0..14]->NXM_NX_TUN_METADATA0[16..30],output:"ovn-ba702e-0",resubmit(,33)
 cookie=0x0, duration=279153.636s, table=32, n_packets=116, n_bytes=10048, priority=0 actions=resubmit(,33)

 ### 30、table33，nat部分，sw发向 vm的流，设置dnat、snat、ct的zone，发往router以及router本身的流量都没有设置ct zone
 ### 31、sw中，单播流量next到table34。广播流量，复制到所有的vm接口（经过table34检查是否是合法同接口进出后才会真正发出[33]）发送，
 cookie=0x0, duration=96240.250s, table=33, n_packets=29, n_bytes=2562, priority=100,reg15=0x1,metadata=0x5 actions=load:0x1->NXM_NX_REG13[],load:0x2->NXM_NX_REG11[],load:0x3->NXM_NX_REG12[],resubmit(,34)
 cookie=0x0, duration=96240.249s, table=33, n_packets=16, n_bytes=1312, priority=100,reg15=0xffff,metadata=0x5 actions=load:0x1->NXM_NX_REG13[],load:0x1->NXM_NX_REG15[],resubmit(,34),load:0x4->NXM_NX_REG13[],load:0x2->NXM_NX_REG15[],resubmit(,34),load:0xffff->NXM_NX_REG15[]
 cookie=0x0, duration=96239.640s, table=33, n_packets=27, n_bytes=2478, priority=100,reg15=0x2,metadata=0x5 actions=load:0x4->NXM_NX_REG13[],load:0x2->NXM_NX_REG11[],load:0x3->NXM_NX_REG12[],resubmit(,34)
 cookie=0x0, duration=87681.565s, table=33, n_packets=27, n_bytes=2590, priority=100,reg15=0x1,metadata=0x6 actions=load:0x5->NXM_NX_REG13[],load:0x6->NXM_NX_REG11[],load:0x7->NXM_NX_REG12[],resubmit(,34)
 cookie=0x0, duration=87681.562s, table=33, n_packets=0, n_bytes=0, priority=100,reg15=0xffff,metadata=0x6 actions=load:0x5->NXM_NX_REG13[],load:0x1->NXM_NX_REG15[],resubmit(,34),load:0xffff->NXM_NX_REG15[]
 cookie=0x0, duration=79156.688s, table=33, n_packets=0, n_bytes=0, priority=100,reg15=0x1,metadata=0x7 actions=load:0x8->NXM_NX_REG11[],load:0x9->NXM_NX_REG12[],resubmit(,34)

 ### 这两个流表完全一样？？？
 cookie=0x0, duration=132318.613s, table=33, n_packets=7, n_bytes=630, priority=100,reg15=0x3,metadata=0x5 actions=load:0x2->NXM_NX_REG11[],load:0x3->NXM_NX_REG12[],resubmit(,34)
 cookie=0x0, duration=79156.688s, table=33, n_packets=0, n_bytes=0,    priority=100,reg15=0x3,metadata=0x5 actions=load:0x2->NXM_NX_REG11[],load:0x3->NXM_NX_REG12[],resubmit(,34)
 cookie=0x0, duration=79156.670s, table=33, n_packets=0, n_bytes=0, priority=100,   reg15=0x3,metadata=0x6 actions=load:0x6->NXM_NX_REG11[],load:0x7->NXM_NX_REG12[],resubmit(,34)


 ### 32、除了打标记的代答报文等，能够从入接口发包，其他drop
 cookie=0x0, duration=96240.250s, table=34, n_packets=8, n_bytes=656, priority=100,reg10=0/0x1,reg14=0x1,reg15=0x1,metadata=0x5 actions=drop
 cookie=0x0, duration=96239.640s, table=34, n_packets=8, n_bytes=656, priority=100,reg10=0/0x1,reg14=0x2,reg15=0x2,metadata=0x5 actions=drop
 cookie=0x0, duration=87681.565s, table=34, n_packets=0, n_bytes=0, priority=100,reg10=0/0x1,reg14=0x1,reg15=0x1,metadata=0x6 actions=drop
 cookie=0x0, duration=79156.688s, table=34, n_packets=0, n_bytes=0, priority=100,reg10=0/0x1,reg14=0x1,reg15=0x1,metadata=0x7 actions=drop
 cookie=0x0, duration=79156.688s, table=34, n_packets=0, n_bytes=0, priority=100,reg10=0/0x1,reg14=0x3,reg15=0x3,metadata=0x5 actions=drop
 cookie=0x0, duration=132318.595s, table=34, n_packets=12, n_bytes=504, priority=100,reg10=0/0x1,reg14=0x3,reg15=0x3,metadata=0x6 actions=drop
 cookie=0x0, duration=79156.670s, table=34, n_packets=0, n_bytes=0, priority=100,reg10=0/0x1,reg14=0x3,reg15=0x3,metadata=0x6 actions=drop
 ### 33、没有drop的，在这里清空了0～9寄存器
 cookie=0x0, duration=279153.636s, table=34, n_packets=144, n_bytes=12680, priority=0 actions=load:0->NXM_NX_REG0[],load:0->NXM_NX_REG1[],load:0->NXM_NX_REG2[],load:0->NXM_NX_REG3[],load:0->NXM_NX_REG4[],load:0->NXM_NX_REG5[],load:0->NXM_NX_REG6[],load:0->NXM_NX_REG7[],load:0->NXM_NX_REG8[],load:0->NXM_NX_REG9[],resubmit(,40)


 cookie=0xa9ce7962, duration=96240.249s, table=40, n_packets=64, n_bytes=5760, priority=0,metadata=0x5 actions=resubmit(,41)
 cookie=0x52b09533, duration=87681.563s, table=40, n_packets=27, n_bytes=2590, priority=0,metadata=0x6 actions=resubmit(,41)
 cookie=0x6d6f641e, duration=79156.683s, table=40, n_packets=0, n_bytes=0, priority=0,metadata=0x7 actions=resubmit(,41)
 cookie=0x994d4ef5, duration=96240.249s, table=41, n_packets=72, n_bytes=6352, priority=0,metadata=0x5 actions=resubmit(,42)
 cookie=0xdd4931f, duration=87681.563s, table=41, n_packets=27, n_bytes=2590, priority=0,metadata=0x6 actions=resubmit(,42)
 cookie=0x8ab720d1, duration=79156.683s, table=41, n_packets=0, n_bytes=0, priority=0,metadata=0x7 actions=resubmit(,42)

 ### reg0=0x1/0x1暂时没用到，后面配置nat的时候应该会涉及
 cookie=0xa72ee3d0, duration=96240.248s, table=42, n_packets=0, n_bytes=0, priority=100,ip,reg0=0x1/0x1,metadata=0x5 actions=ct(table=43,zone=NXM_NX_REG13[0..15])
 cookie=0xeba5cabc, duration=87681.563s, table=42, n_packets=0, n_bytes=0, priority=100,ip,reg0=0x1/0x1,metadata=0x6 actions=ct(table=43,zone=NXM_NX_REG13[0..15])
 cookie=0x9a697c9c, duration=96240.249s, table=42, n_packets=72, n_bytes=6352, priority=0,metadata=0x5 actions=resubmit(,43)
 cookie=0x13706991, duration=87681.563s, table=42, n_packets=27, n_bytes=2590, priority=0,metadata=0x6 actions=resubmit(,43)
 cookie=0x20cb6482, duration=79156.683s, table=42, n_packets=0, n_bytes=0, priority=0,metadata=0x7 actions=resubmit(,43)

 ### 34、router上，发送流程直接next 到64，都是ct流程
 cookie=0x7ad6517b, duration=79156.683s, table=43, n_packets=0, n_bytes=0, priority=100,reg15=0x1,metadata=0x7 actions=resubmit(,64)
 cookie=0x98f146e0, duration=79156.683s, table=43, n_packets=0, n_bytes=0, priority=100,reg15=0x2,metadata=0x7 actions=resubmit(,64)
 cookie=0x68cab2f4, duration=96240.249s, table=43, n_packets=72, n_bytes=6352, priority=0,metadata=0x5 actions=resubmit(,44)
 cookie=0xa5308219, duration=87681.563s, table=43, n_packets=27, n_bytes=2590, priority=0,metadata=0x6 actions=resubmit(,44)
 cookie=0x8f1c32ec, duration=96240.249s, table=44, n_packets=72, n_bytes=6352, priority=0,metadata=0x5 actions=resubmit(,45)
 cookie=0xc5868361, duration=87681.562s, table=44, n_packets=27, n_bytes=2590, priority=0,metadata=0x6 actions=resubmit(,45)
 cookie=0x50958557, duration=96240.249s, table=45, n_packets=72, n_bytes=6352, priority=0,metadata=0x5 actions=resubmit(,46)
 cookie=0x99f2218d, duration=87681.563s, table=45, n_packets=27, n_bytes=2590, priority=0,metadata=0x6 actions=resubmit(,46)
 cookie=0x8f5a5285, duration=96240.248s, table=46, n_packets=72, n_bytes=6352, priority=0,metadata=0x5 actions=resubmit(,47)
 cookie=0x907800bc, duration=87681.562s, table=46, n_packets=27, n_bytes=2590, priority=0,metadata=0x6 actions=resubmit(,47)

 ### 35、nat流程后续补充（reg0未设置）
 cookie=0x3520b159, duration=96240.249s, table=47, n_packets=0, n_bytes=0, priority=100,ip,reg0=0x4/0x4,metadata=0x5 actions=ct(table=48,zone=NXM_NX_REG13[0..15],nat)
 cookie=0xfe7abc9d, duration=87681.563s, table=47, n_packets=0, n_bytes=0, priority=100,ip,reg0=0x4/0x4,metadata=0x6 actions=ct(table=48,zone=NXM_NX_REG13[0..15],nat)
 cookie=0xbb7c7ac2, duration=96240.249s, table=47, n_packets=0, n_bytes=0, priority=100,ip,reg0=0x2/0x2,metadata=0x5 actions=ct(commit,zone=NXM_NX_REG13[0..15],exec(load:0->NXM_NX_CT_LABEL[0])),resubmit(,48)
 cookie=0xffe556b2, duration=87681.563s, table=47, n_packets=0, n_bytes=0, priority=100,ip,reg0=0x2/0x2,metadata=0x6 actions=ct(commit,zone=NXM_NX_REG13[0..15],exec(load:0->NXM_NX_CT_LABEL[0])),resubmit(,48)
 cookie=0xbec85c39, duration=96240.249s, table=47, n_packets=72, n_bytes=6352, priority=0,metadata=0x5 actions=resubmit(,48)
 cookie=0xccc89929, duration=87681.563s, table=47, n_packets=27, n_bytes=2590, priority=0,metadata=0x6 actions=resubmit(,48)

 ### 36、这里应该是port-security 功能触发的流表，对流入vm的 目的mac+ip做检查，流出是在 [3]中完成
 cookie=0xc44012d6, duration=88638.930s, table=48, n_packets=0, n_bytes=0, priority=90,ip,reg15=0x1,metadata=0x5,dl_dst=fa:10:dd:1b:30:01,nw_dst=30.1.1.11 actions=resubmit(,49)
 cookie=0xc44012d6, duration=88638.930s, table=48, n_packets=0, n_bytes=0, priority=90,ip,reg15=0x1,metadata=0x5,dl_dst=fa:10:dd:1b:30:01,nw_dst=30.1.1.255 actions=resubmit(,49)
 cookie=0xc44012d6, duration=88638.930s, table=48, n_packets=0, n_bytes=0, priority=90,ip,reg15=0x1,metadata=0x5,dl_dst=fa:10:dd:1b:30:01,nw_dst=255.255.255.255 actions=resubmit(,49)
 cookie=0xaa43bcf0, duration=88621.461s, table=48, n_packets=0, n_bytes=0, priority=90,ip,reg15=0x2,metadata=0x5,dl_dst=fa:10:dd:1b:30:02,nw_dst=30.1.1.12 actions=resubmit(,49)
 cookie=0xaa43bcf0, duration=88621.461s, table=48, n_packets=0, n_bytes=0, priority=90,ip,reg15=0x2,metadata=0x5,dl_dst=fa:10:dd:1b:30:02,nw_dst=30.1.1.255 actions=resubmit(,49)
 cookie=0xaa43bcf0, duration=88621.461s, table=48, n_packets=0, n_bytes=0, priority=90,ip,reg15=0x2,metadata=0x5,dl_dst=fa:10:dd:1b:30:02,nw_dst=255.255.255.255 actions=resubmit(,49)
 cookie=0xa0296c2b, duration=87681.563s, table=48, n_packets=26, n_bytes=2548, priority=90,ip,reg15=0x1,metadata=0x6,dl_dst=fa:10:dd:1b:40:01,nw_dst=40.1.1.11 actions=resubmit(,49)
 cookie=0xa0296c2b, duration=87681.563s, table=48, n_packets=0, n_bytes=0, priority=90,ip,reg15=0x1,metadata=0x6,dl_dst=fa:10:dd:1b:40:01,nw_dst=255.255.255.255 actions=resubmit(,49)
 cookie=0xa0296c2b, duration=87681.562s, table=48, n_packets=0, n_bytes=0, priority=90,ip,reg15=0x1,metadata=0x6,dl_dst=fa:10:dd:1b:40:01,nw_dst=40.1.1.255 actions=resubmit(,49)
 cookie=0xc44012d6, duration=88638.930s, table=48, n_packets=0, n_bytes=0, priority=90,ip,reg15=0x1,metadata=0x5,dl_dst=fa:10:dd:1b:30:01,nw_dst=224.0.0.0/4 actions=resubmit(,49)
 cookie=0xaa43bcf0, duration=88621.461s, table=48, n_packets=0, n_bytes=0, priority=90,ip,reg15=0x2,metadata=0x5,dl_dst=fa:10:dd:1b:30:02,nw_dst=224.0.0.0/4 actions=resubmit(,49)
 cookie=0xa0296c2b, duration=87681.562s, table=48, n_packets=0, n_bytes=0, priority=90,ip,reg15=0x1,metadata=0x6,dl_dst=fa:10:dd:1b:40:01,nw_dst=224.0.0.0/4 actions=resubmit(,49)
 cookie=0x5aba29f3, duration=88638.930s, table=48, n_packets=0, n_bytes=0, priority=80,ip,reg15=0x1,metadata=0x5,dl_dst=fa:10:dd:1b:30:01 actions=drop
 cookie=0x9ee46441, duration=88621.461s, table=48, n_packets=0, n_bytes=0, priority=80,ip,reg15=0x2,metadata=0x5,dl_dst=fa:10:dd:1b:30:02 actions=drop
 cookie=0x8545cdd2, duration=87681.562s, table=48, n_packets=0, n_bytes=0, priority=80,ip,reg15=0x1,metadata=0x6,dl_dst=fa:10:dd:1b:40:01 actions=drop
 cookie=0xb346c480, duration=96240.249s, table=48, n_packets=72, n_bytes=6352, priority=0,metadata=0x5 actions=resubmit(,49)
 cookie=0xf7f294f, duration=87681.563s, table=48, n_packets=1, n_bytes=42, priority=0,metadata=0x6 actions=resubmit(,49)

 cookie=0xf47b1c4f, duration=96240.248s, table=49, n_packets=16, n_bytes=1312, priority=100,metadata=0x5,dl_dst=01:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,64)
 cookie=0xcfd2ac11, duration=87681.563s, table=49, n_packets=0, n_bytes=0, priority=100,metadata=0x6,dl_dst=01:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,64)
 cookie=0x9ac968ff, duration=88638.930s, table=49, n_packets=0, n_bytes=0, priority=50,reg15=0x1,metadata=0x5,dl_dst=fa:10:dd:1b:30:01 actions=resubmit(,64)
 cookie=0xb72767c4, duration=88621.461s, table=49, n_packets=0, n_bytes=0, priority=50,reg15=0x2,metadata=0x5,dl_dst=fa:10:dd:1b:30:02 actions=resubmit(,64)
 cookie=0xb50a1d6e, duration=87681.563s, table=49, n_packets=27, n_bytes=2590, priority=50,reg15=0x1,metadata=0x6,dl_dst=fa:10:dd:1b:40:01 actions=resubmit(,64)
 cookie=0x6b1a23ee, duration=79156.688s, table=49, n_packets=0, n_bytes=0, priority=50,reg15=0x3,metadata=0x5 actions=resubmit(,64)
 cookie=0x7155a145, duration=79156.670s, table=49, n_packets=0, n_bytes=0, priority=50,reg15=0x3,metadata=0x6 actions=resubmit(,64)


 ### 37、清空IN_PORT后，转table65发送，恢复IN_PORT
 cookie=0x0, duration=96240.250s, table=64, n_packets=5, n_bytes=210, priority=100,reg10=0x1/0x1,reg15=0x1,metadata=0x5 actions=push:NXM_OF_IN_PORT[],load:0->NXM_OF_IN_PORT[],resubmit(,65),pop:NXM_OF_IN_PORT[]
 cookie=0x0, duration=96239.640s, table=64, n_packets=3, n_bytes=126, priority=100,reg10=0x1/0x1,reg15=0x2,metadata=0x5 actions=push:NXM_OF_IN_PORT[],load:0->NXM_OF_IN_PORT[],resubmit(,65),pop:NXM_OF_IN_PORT[]
 cookie=0x0, duration=87681.565s, table=64, n_packets=1, n_bytes=42, priority=100,reg10=0x1/0x1,reg15=0x1,metadata=0x6 actions=push:NXM_OF_IN_PORT[],load:0->NXM_OF_IN_PORT[],resubmit(,65),pop:NXM_OF_IN_PORT[]
 cookie=0x0, duration=79156.688s, table=64, n_packets=0, n_bytes=0, priority=100,reg10=0x1/0x1,reg15=0x3,metadata=0x5 actions=push:NXM_OF_IN_PORT[],load:0->NXM_OF_IN_PORT[],resubmit(,65),pop:NXM_OF_IN_PORT[]
 cookie=0x0, duration=79156.688s, table=64, n_packets=0, n_bytes=0, priority=100,reg10=0x1/0x1,reg15=0x1,metadata=0x7 actions=push:NXM_OF_IN_PORT[],load:0->NXM_OF_IN_PORT[],resubmit(,65),pop:NXM_OF_IN_PORT[]
 cookie=0x0, duration=79156.670s, table=64, n_packets=0, n_bytes=0, priority=100,reg10=0x1/0x1,reg15=0x3,metadata=0x6 actions=push:NXM_OF_IN_PORT[],load:0->NXM_OF_IN_PORT[],resubmit(,65),pop:NXM_OF_IN_PORT[]
 cookie=0x0, duration=195.496s, table=64, n_packets=3, n_bytes=238, priority=100,reg10=0x1/0x1,reg15=0x2,metadata=0x7 actions=push:NXM_OF_IN_PORT[],load:0->NXM_OF_IN_PORT[],resubmit(,65),pop:NXM_OF_IN_PORT[]
 cookie=0x0, duration=279153.636s, table=64, n_packets=120, n_bytes=11056, priority=0 actions=resubmit(,65)

 ### 38、table65就是所有跑到终点的报文的终点站，根据datapath+outport标识，从对应ovs接口发出，注意这里都是通往本机器vm的流量，去往其他主机的都在 [28][29]中从tunnel口发出了
 cookie=0x0, duration=96240.250s, table=65, n_packets=37, n_bytes=3218, priority=100,reg15=0x1,metadata=0x5 actions=output:"sw-300-port-vm1"
 cookie=0x0, duration=96239.640s, table=65, n_packets=35, n_bytes=3134, priority=100,reg15=0x2,metadata=0x5 actions=output:"sw-300-port-vm2"
 cookie=0x0, duration=87681.565s, table=65, n_packets=27, n_bytes=2590, priority=100,reg15=0x1,metadata=0x6 actions=output:"sw-400-port-vm1"
 ### 测试nat的时候补充
  cookie=0x0, duration=213968.582s, table=65, n_packets=9, n_bytes=714, priority=100,reg15=0x3,metadata=0x5 actions=clone(ct_clear,load:0->NXM_NX_REG11[],load:0->NXM_NX_REG12[],load:0->NXM_NX_REG13[],load:0x8->NXM_NX_REG11[],load:0x9->NXM_NX_REG12[],load:0x7->OXM_OF_METADATA[],load:0x1->NXM_NX_REG14[],load:0->NXM_NX_REG10[],load:0->NXM_NX_REG15[],load:0->NXM_NX_REG0[],load:0->NXM_NX_REG1[],load:0->NXM_NX_REG2[],load:0->NXM_NX_REG3[],load:0->NXM_NX_REG4[],load:0->NXM_NX_REG5[],load:0->NXM_NX_REG6[],load:0->NXM_NX_REG7[],load:0->NXM_NX_REG8[],load:0->NXM_NX_REG9[],load:0->NXM_OF_IN_PORT[],resubmit(,8))
 cookie=0x0, duration=213968.582s, table=65, n_packets=9, n_bytes=714, priority=100,reg15=0x1,metadata=0x7 actions=clone(ct_clear,load:0->NXM_NX_REG11[],load:0->NXM_NX_REG12[],load:0->NXM_NX_REG13[],load:0x2->NXM_NX_REG11[],load:0x3->NXM_NX_REG12[],load:0x5->OXM_OF_METADATA[],load:0x3->NXM_NX_REG14[],load:0->NXM_NX_REG10[],load:0->NXM_NX_REG15[],load:0->NXM_NX_REG0[],load:0->NXM_NX_REG1[],load:0->NXM_NX_REG2[],load:0->NXM_NX_REG3[],load:0->NXM_NX_REG4[],load:0->NXM_NX_REG5[],load:0->NXM_NX_REG6[],load:0->NXM_NX_REG7[],load:0->NXM_NX_REG8[],load:0->NXM_NX_REG9[],load:0->NXM_OF_IN_PORT[],resubmit(,8))
 cookie=0x0, duration=213968.564s, table=65, n_packets=15, n_bytes=742, priority=100,reg15=0x3,metadata=0x6 actions=clone(ct_clear,load:0->NXM_NX_REG11[],load:0->NXM_NX_REG12[],load:0->NXM_NX_REG13[],load:0x8->NXM_NX_REG11[],load:0x9->NXM_NX_REG12[],load:0x7->OXM_OF_METADATA[],load:0x2->NXM_NX_REG14[],load:0->NXM_NX_REG10[],load:0->NXM_NX_REG15[],load:0->NXM_NX_REG0[],load:0->NXM_NX_REG1[],load:0->NXM_NX_REG2[],load:0->NXM_NX_REG3[],load:0->NXM_NX_REG4[],load:0->NXM_NX_REG5[],load:0->NXM_NX_REG6[],load:0->NXM_NX_REG7[],load:0->NXM_NX_REG8[],load:0->NXM_NX_REG9[],load:0->NXM_OF_IN_PORT[],resubmit(,8))
 cookie=0x0, duration=81845.465s, table=65, n_packets=3, n_bytes=238, priority=100,reg15=0x2,metadata=0x7 actions=clone(ct_clear,load:0->NXM_NX_REG11[],load:0->NXM_NX_REG12[],load:0->NXM_NX_REG13[],load:0x6->NXM_NX_REG11[],load:0x7->NXM_NX_REG12[],load:0x6->OXM_OF_METADATA[],load:0x3->NXM_NX_REG14[],load:0->NXM_NX_REG10[],load:0->NXM_NX_REG15[],load:0->NXM_NX_REG0[],load:0->NXM_NX_REG1[],load:0->NXM_NX_REG2[],load:0->NXM_NX_REG3[],load:0->NXM_NX_REG4[],load:0->NXM_NX_REG5[],load:0->NXM_NX_REG6[],load:0->NXM_NX_REG7[],load:0->NXM_NX_REG8[],load:0->NXM_NX_REG9[],load:0->NXM_OF_IN_PORT[],resubmit(,8))

 ### 39、作用是为非系统管理nexthop修改下一跳mac， 见[5]，mac地址学习流程。
 cookie=0x0, duration=679.005s, table=66, n_packets=0, n_bytes=0, priority=100,reg0=0x1e01010b,reg15=0x1,metadata=0x7 actions=mod_dl_dst:fa:10:dd:1b:30:01
 cookie=0x0, duration=650.303s, table=66, n_packets=0, n_bytes=0, priority=100,reg0=0x1e01010c,reg15=0x1,metadata=0x7 actions=mod_dl_dst:fa:10:dd:1b:30:02
 cookie=0x0, duration=157.431s, table=66, n_packets=0, n_bytes=0, priority=100,reg0=0x2801010b,reg15=0x2,metadata=0x7 actions=mod_dl_dst:fa:10:dd:1b:40:01
```

# 访问公网

```
### 确认为转发后，进入table15，其中需要注意的是 NXM_OF_IP_DST[]->NXM_NX_XXREG0[96..127]将下一跳存入reg0 供后续流表使用。
### 1、metadata=0x7表示VPC路由器，路由器上的三个直连网段的转发，其中两个是VPC网络，一个是公网网络
##     转发对报文的修改：src mac=网关mac，ttl--，和内核协议栈处理类似。
 cookie=0xcf105cd3, duration=13247.735s, table=15, n_packets=0, n_bytes=0, priority=49,ip,metadata=0x7,nw_dst=40.1.1.0/24 actions=dec_ttl(),move:NXM_OF_IP_DST[]->NXM_NX_XXREG0[96..127],load:0x28010101->NXM_NX_XXREG0[64..95],mod_dl_src:02:d4:1d:8c:40:01,load:0x2->NXM_NX_REG15[],load:0x1->NXM_NX_REG10[0],resubmit(,16)
 cookie=0x448ecdcd, duration=13247.733s, table=15, n_packets=15, n_bytes=1470, priority=49,ip,metadata=0x7,nw_dst=30.1.1.0/24 actions=dec_ttl(),move:NXM_OF_IP_DST[]->NXM_NX_XXREG0[96..127],load:0x1e010101->NXM_NX_XXREG0[64..95],mod_dl_src:02:d4:1d:8c:30:01,load:0x1->NXM_NX_REG15[],load:0x1->NXM_NX_REG10[0],resubmit(,16)
 cookie=0x74fa48c4, duration=3298.141s, table=15, n_packets=275, n_bytes=26950, priority=49,ip,metadata=0x7,nw_dst=192.168.77.0/24 actions=dec_ttl(),move:NXM_OF_IP_DST[]->NXM_NX_XXREG0[96..127],load:0xc0a84d01->NXM_NX_XXREG0[64..95],mod_dl_src:02:d4:1d:8c:ff:01,load:0x3->NXM_NX_REG15[],load:0x1->NXM_NX_REG10[0],resubmit(,16)
### 2、默认路由触发的流表，在logic router如果没匹配上面三条直连路由流表，则走默认路由，下一跳是 192.168.77.254，
##    下一跳也是存在reg0中，修改smac、ttl
 cookie=0x8a2f7740, duration=1565.280s, table=15, n_packets=2, n_bytes=196, priority=1,ip,metadata=0x7 actions=dec_ttl(),load:0xc0a84dfe->NXM_NX_XXREG0[96..127],load:0xc0a84d01->NXM_NX_XXREG0[64..95],mod_dl_src:02:d4:1d:8c:ff:01,load:0x3->NXM_NX_REG15[],load:0x1->NXM_NX_REG10[0],resubmit(,16)

### 3、这个流表resubmit了两次，第一次到table66，是为了刷dmac，这样加上上面table15中修改了smac和ttl，三层转发流程完成。
##  然后next table17走output流程。table66是动态table，是控制器通过mac地址学习动态下发的。见上一篇中的mac地址学习流程。
##     
 cookie=0xd5478283, duration=13247.733s, table=16, n_packets=292, n_bytes=28616, priority=0,ip,metadata=0x7 actions=push:NXM_NX_REG0[],push:NXM_NX_XXREG0[96..127],pop:NXM_NX_REG0[],mod_dl_dst:00:00:00:00:00:00,resubmit(,66),pop:NXM_NX_REG0[],resubmit(,17)

 cookie=0x0, duration=1599.369s, table=66, n_packets=3, n_bytes=294, priority=100,reg0=0xc0a84dfe,reg15=0x3,metadata=0x7 actions=mod_dl_dst:2e:bb:8d:e4:2d:bd


 ### 这里贴出了上一节中table66流表格式。
 cookie=0x0, duration=679.005s, table=66, n_packets=0, n_bytes=0, priority=100,reg0=0x1e01010b,reg15=0x1,metadata=0x7 actions=mod_dl_dst:fa:10:dd:1b:30:01
 cookie=0x0, duration=650.303s, table=66, n_packets=0, n_bytes=0, priority=100,reg0=0x1e01010c,reg15=0x1,metadata=0x7 actions=mod_dl_dst:fa:10:dd:1b:30:02
 cookie=0x0, duration=157.431s, table=66, n_packets=0, n_bytes=0, priority=100,reg0=0x2801010b,reg15=0x2,metadata=0x7 actions=mod_dl_dst:fa:10:dd:1b:40:01


###sw-pub流表
cookie=0x107a5839, duration=10868.011s, table=24, n_packets=672, n_bytes=64904, idle_age=3, priority=0,metadata=0x8 actions=load:0xfffe->NXM_NX_REG15[],resubmit(,32)
cookie=0x0, duration=10868.011s, table=33, n_packets=672, n_bytes=64904, idle_age=3, hard_age=10867, priority=100,reg15=0xfffe,metadata=0x8 actions=load:0xc->NXM_NX_REG13[],load:0x1->NXM_NX_REG15[],resubmit(,34),load:0xfffe->NXM_NX_REG15[]
cookie=0x0, duration=10867.958s, table=65, n_packets=676, n_bytes=65072, idle_age=3, priority=100,reg15=0x1,metadata=0x8 actions=output:"patch-br-int-to"
```

# NAT

```
ovn-nbctl lr-nat-add vpc-router snat 192.168.77.1 30.1.1.0/24

### 1、路由器上，目的是192.168.77.1的包，进入ct处理，如果是公网进入云网络的包，此前已经在出方向建立了est的ct表项，这里就能够做dnat，
##    将公网地址192.168.77.1转化内网地址发往vm
 cookie=0x7b8e0e0e, duration=169.183s, table=11, n_packets=0, n_bytes=0, priority=100,ip,reg14=0x3,metadata=0x7,nw_dst=192.168.77.1 actions=ct(table=12,zone=NXM_NX_REG12[0..15],nat)

 cookie=0x7580a142, duration=169.183s, table=11, n_packets=0, n_bytes=0, priority=50,ip,metadata=0x7,nw_dst=192.168.77.1 actions=load:0x1->OXM_OF_PKT_REG4[0],resubmit(,12)
### 2、router上，出公网流量，确认从公网口出后，为VPC网络 30.1.1.0/24建立 snat conntrack表项
 cookie=0x1939fcab, duration=169.183s, table=41, n_packets=0, n_bytes=0, priority=25,ip,reg15=0x3,metadata=0x7,nw_src=30.1.1.0/24 actions=ct(commit,table=42,zone=NXM_NX_REG12[0..15],nat(src=192.168.77.1))
### 
 cookie=0xadd21e5a, duration=169.183s, table=42, n_packets=0, n_bytes=0, priority=100,ip,reg15=0x3,metadata=0x7,nw_dst=192.168.77.1 actions=clone(ct_clear,move:NXM_NX_REG15[]->NXM_NX_REG14[],load:0->NXM_NX_REG15[],load:0->NXM_NX_REG10[],load:0x1->NXM_NX_REG10[0],load:0->NXM_NX_XXREG0[96..127],load:0->NXM_NX_XXREG0[64..95],load:0->NXM_NX_XXREG0[32..63],load:0->NXM_NX_XXREG0[0..31],load:0->NXM_NX_XXREG1[96..127],load:0->NXM_NX_XXREG1[64..95],load:0->NXM_NX_XXREG1[32..63],load:0->NXM_NX_XXREG1[0..31],load:0->OXM_OF_PKT_REG4[32..63],load:0->OXM_OF_PKT_REG4[0..31],load:0x1->OXM_OF_PKT_REG4[1],resubmit(,8))
```

# EIP

```
ovn-nbctl lr-nat-add vpc-router dnat_and_snat 192.168.77.32 30.1.1.12 sw-300-port-vm2 0a:10:dd:1b:30:02
ovn-nbctl lr-nat-add vpc-router dnat_and_snat 192.168.77.42 40.1.1.12 sw-400-port-vm2  0a:10:dd:1b:40:02

注意，如果要实现分布式EIP的特性，需要在每个节点上都配置公网桥。
ovs-vsctl add-br br-ex
ovs-vsctl set Open_vSwitch . external-ids:ovn-bridge-mappings=dataNet:br-ex

###### 本节点为Centrial，sw-300-port-vm2（0a:10:dd:1b:30:02）在Centrial节点，
##     sw-400-port-vm2（0a:10:dd:1b:40:02） 在Node节点，重点关注nat和分布式EIP特性

#### 1、只收dmac 为 0a:10:dd:1b:30:02 的报文，0a:10:dd:1b:40:02 在Node节点接收
 cookie=0x34dc7936, duration=959.144s, table=8, n_packets=0, n_bytes=0, priority=50,reg14=0x3,metadata=0x7,dl_dst=0a:10:dd:1b:30:02 actions=resubmit(,9)

#### 2、公网进入的arp报文代答流表，使用 EXTERNAL_IP的mac地址应答
#.   192.168.77.32（本机IP）的arp请求，应答来自VPC网络和公网网络的请求
#.   192.168.77.42（非本机）的arp请求，只应答来自VPC网络的请求，公网侧必须在 logic ip所在宿主机Node节点应答，
#    否则可能在公网 sw 上发生mac地址漂移，流量异常。
####
 cookie=0xb1db9759, duration=1712.735s, table=9, n_packets=0, n_bytes=0, priority=90,arp,reg14=0x1,metadata=0x7,arp_tpa=192.168.77.32,arp_op=1 actions=move:NXM_OF_ETH_SRC[]->NXM_OF_ETH_DST[],load:0x2->NXM_OF_ARP_OP[],move:NXM_NX_ARP_SHA[]->NXM_NX_ARP_THA[],mod_dl_src:02:d4:1d:8c:30:01,load:0x2d41d8c3001->NXM_NX_ARP_SHA[],move:NXM_OF_ARP_SPA[]->NXM_OF_ARP_TPA[],load:0xc0a84d20->NXM_OF_ARP_SPA[],load:0x1->NXM_NX_REG15[],load:0x1->NXM_NX_REG10[0],resubmit(,32)
 cookie=0xb48b1ba3, duration=1712.735s, table=9, n_packets=0, n_bytes=0, priority=90,arp,reg14=0x2,metadata=0x7,arp_tpa=192.168.77.32,arp_op=1 actions=move:NXM_OF_ETH_SRC[]->NXM_OF_ETH_DST[],load:0x2->NXM_OF_ARP_OP[],move:NXM_NX_ARP_SHA[]->NXM_NX_ARP_THA[],mod_dl_src:02:d4:1d:8c:40:01,load:0x2d41d8c4001->NXM_NX_ARP_SHA[],move:NXM_OF_ARP_SPA[]->NXM_OF_ARP_TPA[],load:0xc0a84d20->NXM_OF_ARP_SPA[],load:0x2->NXM_NX_REG15[],load:0x1->NXM_NX_REG10[0],resubmit(,32)
 cookie=0x4b6c758e, duration=1712.735s, table=9, n_packets=1, n_bytes=42, priority=90,arp,reg14=0x3,metadata=0x7,arp_tpa=192.168.77.32,arp_op=1 actions=move:NXM_OF_ETH_SRC[]->NXM_OF_ETH_DST[],load:0x2->NXM_OF_ARP_OP[],move:NXM_NX_ARP_SHA[]->NXM_NX_ARP_THA[],mod_dl_src:0a:10:dd:1b:30:02,load:0xa10dd1b3002->NXM_NX_ARP_SHA[],move:NXM_OF_ARP_SPA[]->NXM_OF_ARP_TPA[],load:0xc0a84d20->NXM_OF_ARP_SPA[],load:0x3->NXM_NX_REG15[],load:0x1->NXM_NX_REG10[0],resubmit(,32)

 cookie=0xbad18a69, duration=393.564s, table=9, n_packets=0, n_bytes=0, priority=90,arp,reg14=0x2,metadata=0x7,arp_tpa=192.168.77.42,arp_op=1   actions=move:NXM_OF_ETH_SRC[]->NXM_OF_ETH_DST[],load:0x2->NXM_OF_ARP_OP[],move:NXM_NX_ARP_SHA[]->NXM_NX_ARP_THA[],mod_dl_src:02:d4:1d:8c:40:01,load:0x2d41d8c4001->NXM_NX_ARP_SHA[],move:NXM_OF_ARP_SPA[]->NXM_OF_ARP_TPA[],load:0xc0a84d2a->NXM_OF_ARP_SPA[],load:0x2->NXM_NX_REG15[],load:0x1->NXM_NX_REG10[0],resubmit(,32)
 cookie=0xe86209f6, duration=393.564s, table=9, n_packets=0, n_bytes=0, priority=90,arp,reg14=0x1,metadata=0x7,arp_tpa=192.168.77.42,arp_op=1   actions=move:NXM_OF_ETH_SRC[]->NXM_OF_ETH_DST[],load:0x2->NXM_OF_ARP_OP[],move:NXM_NX_ARP_SHA[]->NXM_NX_ARP_THA[],mod_dl_src:02:d4:1d:8c:30:01,load:0x2d41d8c3001->NXM_NX_ARP_SHA[],move:NXM_OF_ARP_SPA[]->NXM_OF_ARP_TPA[],load:0xc0a84d2a->NXM_OF_ARP_SPA[],load:0x1->NXM_NX_REG15[],load:0x1->NXM_NX_REG10[0],resubmit(,32)

#### 3、路由器上到 EIP 的流量，进入CT模块做nat处理，如果已经存在ct表项，则做完dnat，进入table12直接next table13。否则在table12中做dnat
 cookie=0xfc13ddd1, duration=959.144s, table=11, n_packets=0, n_bytes=0, priority=100,ip,reg14=0x3,metadata=0x7,nw_dst=192.168.77.32 actions=ct(table=12,zone=NXM_NX_REG12[0..15],nat)
 cookie=0x341855db, duration=393.564s, table=11, n_packets=0, n_bytes=0, priority=100,ip,reg14=0x3,metadata=0x7,nw_dst=192.168.77.42 actions=ct(table=12,zone=NXM_NX_REG12[0..15],nat)
 cookie=0x35164f7c, duration=959.144s, table=11, n_packets=0, n_bytes=0, priority=50,ip,metadata=0x7,nw_dst=192.168.77.32 actions=load:0x1->OXM_OF_PKT_REG4[0],resubmit(,12)
 cookie=0x1b958ca2, duration=393.564s, table=11, n_packets=0, n_bytes=0, priority=50,ip,metadata=0x7,nw_dst=192.168.77.42 actions=load:0x1->OXM_OF_PKT_REG4[0],resubmit(,12)

#### 4、路由器上到 EIP 的流量，如果从公网来，DNAT 为VM IP，并创建CT表项。如果从内网来，为普通到EIP的报文。
 cookie=0x15203869, duration=959.143s, table=12, n_packets=0, n_bytes=0, priority=100,ip,reg14=0x3,metadata=0x7,nw_dst=192.168.77.32 actions=ct(commit,table=13,zone=NXM_NX_REG11[0..15],nat(dst=30.1.1.12))
 cookie=0x2c74407a, duration=393.564s, table=12, n_packets=0, n_bytes=0, priority=100,ip,reg14=0x3,metadata=0x7,nw_dst=192.168.77.42 actions=ct(commit,table=13,zone=NXM_NX_REG11[0..15],nat(dst=40.1.1.12))
 cookie=0x4ee4c35c, duration=959.143s, table=12, n_packets=0, n_bytes=0, priority=50,ip,metadata=0x7,nw_dst=192.168.77.32 actions=load:0x1->OXM_OF_PKT_REG4[0],resubmit(,13)
 cookie=0xbfda9be3, duration=393.564s, table=12, n_packets=0, n_bytes=0, priority=50,ip,metadata=0x7,nw_dst=192.168.77.42 actions=load:0x1->OXM_OF_PKT_REG4[0],resubmit(,13)
 cookie=0x2f100e98, duration=99026.437s, table=12, n_packets=1436, n_bytes=140236, priority=0,metadata=0x7 actions=resubmit(,13)

 cookie=0xa616964f, duration=959.144s, table=17, n_packets=2, n_bytes=196, priority=100,ip,reg15=0x3,metadata=0x7,nw_src=30.1.1.12 actions=resubmit(,18)
 cookie=0xcd109cd8, duration=393.564s, table=17, n_packets=0, n_bytes=0, priority=100,ip,reg15=0x3,metadata=0x7,nw_src=40.1.1.12 actions=resubmit(,18)

#### 5、VM->pub的流量，snat处理，走ct表项做nat。第二条实际上是走不到的，n_packets=0,实际流量在Node节点上
 cookie=0x2345dfd5, duration=959.144s, table=40, n_packets=2, n_bytes=196, priority=100,ip,reg15=0x3,metadata=0x7,nw_src=30.1.1.12 actions=mod_dl_src:0a:10:dd:1b:30:02,ct(table=41,zone=NXM_NX_REG11[0..15],nat)
 cookie=0x9fb1af16, duration=393.564s, table=40, n_packets=0, n_bytes=0, priority=100,ip,reg15=0x3,metadata=0x7,nw_src=40.1.1.12 actions=mod_dl_src:0a:10:dd:1b:40:02,ct(table=41,zone=NXM_NX_REG11[0..15],nat)
#### 6、VM->pub的流量，snat处理，nat并建立ct表项
 cookie=0x25fcdcde, duration=959.143s, table=41, n_packets=2, n_bytes=196, priority=33,ip,reg15=0x3,metadata=0x7,nw_src=30.1.1.12 actions=mod_dl_src:0a:10:dd:1b:30:02,ct(commit,table=42,zone=NXM_NX_REG12[0..15],nat(src=192.168.77.32))
 cookie=0x700c42e4, duration=393.564s, table=41, n_packets=0, n_bytes=0, priority=33,ip,reg15=0x3,metadata=0x7,nw_src=40.1.1.12 actions=mod_dl_src:0a:10:dd:1b:40:02,ct(commit,table=42,zone=NXM_NX_REG12[0..15],nat(src=192.168.77.42))

#### 7、举例说明，vm2 ping EIP, "ping 192.168.77.32 -I 30.1.1.12"，到这里变成：
#     0a:10:dd:1b:30:02（上面snat加工） --> 02:d4:1d:8c:30:1 , 192.168.77.32(上面snat加工） --> 192.168.77.32
#     这里先清除在上面建立的 ct表项（这种情况不需要ct），模拟从公网收包：将入接口设置为公网口（当前的出口），然后送到 table8 再走一遍
#     其结果是，经过 dnat等操作，将报文改成 0a:10:dd:1b:30:02 --> fa:10:dd:1b:30:02 , 192.168.77.32 --> 30.1.1.12，送入vm接口，数据包完全正确。
####
 cookie=0x23ac3e3b, duration=959.144s, table=42, n_packets=0, n_bytes=0, priority=100,ip,reg15=0x3,metadata=0x7,nw_dst=192.168.77.32 actions=clone(ct_clear,move:NXM_NX_REG15[]->NXM_NX_REG14[],load:0->NXM_NX_REG15[],load:0->NXM_NX_REG10[],load:0x1->NXM_NX_REG10[0],load:0->NXM_NX_XXREG0[96..127],load:0->NXM_NX_XXREG0[64..95],load:0->NXM_NX_XXREG0[32..63],load:0->NXM_NX_XXREG0[0..31],load:0->NXM_NX_XXREG1[96..127],load:0->NXM_NX_XXREG1[64..95],load:0->NXM_NX_XXREG1[32..63],load:0->NXM_NX_XXREG1[0..31],load:0->OXM_OF_PKT_REG4[32..63],load:0->OXM_OF_PKT_REG4[0..31],load:0x1->OXM_OF_PKT_REG4[1],resubmit(,8))
 cookie=0xe5410abf, duration=393.565s, table=42, n_packets=0, n_bytes=0, priority=100,ip,reg15=0x3,metadata=0x7,nw_dst=192.168.77.42 actions=clone(ct_clear,move:NXM_NX_REG15[]->NXM_NX_REG14[],load:0->NXM_NX_REG15[],load:0->NXM_NX_REG10[],load:0x1->NXM_NX_REG10[0],load:0->NXM_NX_XXREG0[96..127],load:0->NXM_NX_XXREG0[64..95],load:0->NXM_NX_XXREG0[32..63],load:0->NXM_NX_XXREG0[0..31],load:0->NXM_NX_XXREG1[96..127],load:0->NXM_NX_XXREG1[64..95],load:0->NXM_NX_XXREG1[32..63],load:0->NXM_NX_XXREG1[0..31],load:0->OXM_OF_PKT_REG4[32..63],load:0->OXM_OF_PKT_REG4[0..31],load:0x1->OXM_OF_PKT_REG4[1],resubmit(,8))
```

# DHCP

```
### 配置sw子网
ovn-nbctl set logical_switch sw-300 \
  other_config:subnet="30.1.1.0/24" \
  other_config:exclude_ips="30.1.1.1..30.1.1.99"

### 配置dhcp option 
CIDR_UUID300=$(ovn-nbctl create dhcp_options \
  cidr=30.1.1.0/24 \
  options='"lease_time"="3600" "router"="30.1.1.1" "server_id"="30.1.1.1" "server_mac"="c0:ff:ee:00:30:01"')

### vm接口绑定 dhcp option
ovn-nbctl lsp-add sw-300 sw-300-port-vm2
ovn-nbctl lsp-set-addresses sw-300-port-vm2 "fa:10:dd:1b:30:01 dynamic"
ovn-nbctl lsp-set-dhcpv4-options sw-300-port-vm2 $CIDR_UUID300


### 1、dhcp REQUEST
 cookie=0x5d4c7169, duration=1226.116s, table=25, n_packets=0, n_bytes=0, priority=100,udp,reg14=0x2,metadata=0x2,dl_src=fa:10:dd:1b:30:01,nw_src=30.1.1.101,nw_dst=255.255.255.255,tp_src=68,tp_dst=67 actions=controller(userdata=00.00.00.02.00.00.00.00.00.01.de.10.00.00.00.63.1e.01.01.65.33.04.00.00.0e.10.01.04.ff.ff.ff.00.03.04.1e.01.01.01.36.04.1e.01.01.01,pause),resubmit(,26)
 cookie=0x5d4c7169, duration=1226.116s, table=25, n_packets=0, n_bytes=0, priority=100,udp,reg14=0x2,metadata=0x2,dl_src=fa:10:dd:1b:30:01,nw_src=30.1.1.101,nw_dst=30.1.1.1,tp_src=68,tp_dst=67 actions=controller(userdata=00.00.00.02.00.00.00.00.00.01.de.10.00.00.00.63.1e.01.01.65.33.04.00.00.0e.10.01.04.ff.ff.ff.00.03.04.1e.01.01.01.36.04.1e.01.01.01,pause),resubmit(,26)

### 2、dhcp DHCPDISCOVER报文匹配，送控制器处理，控制器做 put_dhcp_opts action，DHCP请求包转换为回复包（OFFER）
 cookie=0xe4d5f032, duration=1226.116s, table=25, n_packets=0, n_bytes=0, priority=100,udp,reg14=0x2,metadata=0x2,dl_src=fa:10:dd:1b:30:01,nw_src=0.0.0.0,nw_dst=255.255.255.255,tp_src=68,tp_dst=67 actions=controller(userdata=00.00.00.02.00.00.00.00.00.01.de.10.00.00.00.63.1e.01.01.65.33.04.00.00.0e.10.01.04.ff.ff.ff.00.03.04.1e.01.01.01.36.04.1e.01.01.01,pause),resubmit(,26)

### 3、所有回复包（OFFER、ACK）的三四层头修改
 cookie=0xa311c4bc, duration=1226.116s, table=26, n_packets=0, n_bytes=0, priority=100,udp,reg0=0x8/0x8,reg14=0x2,metadata=0x2,dl_src=fa:10:dd:1b:30:01,tp_src=68,tp_dst=67 actions=move:NXM_OF_ETH_SRC[]->NXM_OF_ETH_DST[],mod_dl_src:c0:ff:ee:00:30:01,mod_nw_src:30.1.1.1,mod_tp_src:67,mod_tp_dst:68,move:NXM_NX_REG14[]->NXM_NX_REG15[],load:0x1->NXM_NX_REG10[0],resubmit(,37)
```

# 安全组ACL

```
### 默认 drop
ovn-nbctl --name=vm-400-2-in-default  acl-add  sw-400 to-lport 0 'outport == "sw-400-port-vm2" && ip' drop
### 配置 允许30.1.1.12访问 4444端口
ovn-nbctl --name=vm-400-2-in-permit4444 acl-add  sw-400 to-lport 1000 'outport == "sw-400-port-vm2" && ip4.src == 30.1.1.12 && tcp.dst == 4444' allow-related
### 创建ip地址集合
ovn-nbctl create Address_Set name=vm_300_ipset addresses='30.1.1.111 30.1.1.112'
### 使用地址集，允许地址集中的vm访问5555端口
ovn-nbctl  acl-add  sw-400 to-lport 1000 'outport == "sw-400-port-vm2" && ip4.src == $vm_300_ipset && tcp.dst == 5555' allow-related


#### 1、ingress 方向安全组
## tcp rst 和 icmp type3 都是 acl的reject verdict 生成的报文，我们只配置了drop，所以没有实际的流量匹配这些流表
cookie=0x568f4201, duration=4690.495s, table=11, n_packets=0, n_bytes=0, priority=110,tcp,metadata=0x6,tcp_flags=rst actions=resubmit(,12)
 cookie=0x568f4201, duration=4690.496s, table=11, n_packets=0, n_bytes=0, priority=110,icmp,metadata=0x6,icmp_type=3 actions=resubmit(,12)
 cookie=0xf811791a, duration=4690.496s, table=11, n_packets=0, n_bytes=0, priority=110,ip,reg14=0x3,metadata=0x6 actions=resubmit(,12)
 cookie=0x1c982666, duration=4690.496s, table=11, n_packets=9, n_bytes=706, priority=100,ip,metadata=0x6 actions=load:0x1->NXM_NX_XXREG0[96],resubmit(,12)

 cookie=0x7ff1837, duration=4690.496s, table=14, n_packets=0, n_bytes=0, priority=65535,ct_state=-new-est+rel-inv+trk,ct_label=0/0x1,metadata=0x6 actions=resubmit(,15)
## 已双向跟踪 && 反向流量 && 打了blocked flag == drop
 cookie=0xda49a9a8, duration=4690.496s, table=14, n_packets=0, n_bytes=0, priority=65535,ct_state=+est+rpl+trk,ct_label=0x1/0x1,metadata=0x6 actions=drop
## 无效报文 == drop
 cookie=0xda49a9a8, duration=4690.496s, table=14, n_packets=0, n_bytes=0, priority=65535,ct_state=+inv+trk,metadata=0x6 actions=drop
## "ct.est && !ct.rel && !ct.new && !ct.inv && ct.rpl && ct_label.blocked == 0"
 cookie=0x546d5489, duration=4690.496s, table=14, n_packets=9, n_bytes=706, priority=65535,ct_state=-new+est-rel+rpl-inv+trk,ct_label=0/0x1,metadata=0x6 actions=resubmit(,15)
## "ip && (!ct.est || (ct.est && ct_label.blocked == 1))" == 标记 reg0 = 1
 cookie=0x1349518f, duration=4690.496s, table=14, n_packets=0, n_bytes=0, priority=1,ct_state=-est+trk,ip,metadata=0x6 actions=load:0x1->NXM_NX_XXREG0[97],resubmit(,15)
 cookie=0x1349518f, duration=4690.496s, table=14, n_packets=0, n_bytes=0, priority=1,ct_state=+est+trk,ct_label=0x1/0x1,ip,metadata=0x6 actions=load:0x1->NXM_NX_XXREG0[97],resubmit(,15)

#### 2、egress 方向安全组
 cookie=0x58a88198, duration=4690.496s, table=41, n_packets=0, n_bytes=0, priority=110,icmp,metadata=0x6,icmp_type=3 actions=resubmit(,42)
 cookie=0x58a88198, duration=4690.496s, table=41, n_packets=0, n_bytes=0, priority=110,tcp,metadata=0x6,tcp_flags=rst actions=resubmit(,42)
 cookie=0xe70e698d, duration=4690.496s, table=41, n_packets=9, n_bytes=706, priority=110,ip,reg15=0x3,metadata=0x6 actions=resubmit(,42)
 cookie=0x8a28c202, duration=4690.496s, table=41, n_packets=37, n_bytes=2755, priority=100,ip,metadata=0x6 actions=load:0x1->NXM_NX_XXREG0[96],resubmit(,42)

#### 3、以下resubmit(,45)都是白名单放行的各种情况，其他都会进入 [4] 代表的 default action流表中 执行drop
##  eg, 30.1.1.12 --> 40.1.1.12:4444，会先后走到这里的 91ef48b0（正方向 new），6169800e（正方向 est），a2bddbb0（反方向）放行。
## 流表中所有的控制器操作都是由于我配置了 --name，触发的 log处理，会触发上送控制器打日志。
##
 cookie=0xa2bddbb0, duration=4690.496s, table=44, n_packets=9, n_bytes=706, priority=65535,ct_state=-new+est-rel+rpl-inv+trk,ct_label=0/0x1,metadata=0x6 actions=resubmit(,45)
 cookie=0x628a564d, duration=4690.496s, table=44, n_packets=0, n_bytes=0, priority=65535,ct_state=+est+rpl+trk,ct_label=0x1/0x1,metadata=0x6 actions=drop
 cookie=0xcc4f7527, duration=4690.496s, table=44, n_packets=0, n_bytes=0, priority=65535,ct_state=-new-est+rel-inv+trk,ct_label=0/0x1,metadata=0x6 actions=resubmit(,45)
 cookie=0x628a564d, duration=4690.495s, table=44, n_packets=0, n_bytes=0, priority=65535,ct_state=+inv+trk,metadata=0x6 actions=drop

 cookie=0x91ef48b0, duration=4690.496s, table=44, n_packets=0, n_bytes=0, priority=2000,ct_state=-new+est-rpl+trk,ct_label=0x1/0x1,tcp,reg15=0x2,metadata=0x6,nw_src=30.1.1.12,tp_dst=4444 actions=load:0x1->NXM_NX_XXREG0[97],controller(userdata=00.00.00.07.00.00.00.00.00.06.76.6d.2d.34.30.30.2d.32.2d.69.6e.2d.70.65.72.6d.69.74.34.34.34.34),resubmit(,45)
##
 cookie=0x6169800e, duration=4690.496s, table=44, n_packets=8, n_bytes=537, priority=2000,ct_state=-new+est-rpl+trk,ct_label=0/0x1,tcp,reg15=0x2,metadata=0x6,nw_src=30.1.1.12,tp_dst=4444 actions=controller(userdata=00.00.00.07.00.00.00.00.00.06.76.6d.2d.34.30.30.2d.32.2d.69.6e.2d.70.65.72.6d.69.74.34.34.34.34),resubmit(,45)
 cookie=0x4f257b76, duration=30.827s, table=44, n_packets=0, n_bytes=0, priority=2000,ct_state=-new+est-rpl+trk,ct_label=0x1/0x1,tcp,reg15=0x2,metadata=0x6,nw_src=30.1.1.111,tp_dst=5555 actions=load:0x1->NXM_NX_XXREG0[97],resubmit(,45)
 cookie=0x32d72a94, duration=30.827s, table=44, n_packets=0, n_bytes=0, priority=2000,ct_state=-new+est-rpl+trk,ct_label=0/0x1,tcp,reg15=0x2,metadata=0x6,nw_src=30.1.1.111,tp_dst=5555 actions=resubmit(,45)
 cookie=0x4f257b76, duration=30.827s, table=44, n_packets=0, n_bytes=0, priority=2000,ct_state=-new+est-rpl+trk,ct_label=0x1/0x1,tcp,reg15=0x2,metadata=0x6,nw_src=30.1.1.112,tp_dst=5555 actions=load:0x1->NXM_NX_XXREG0[97],resubmit(,45)
 cookie=0x32d72a94, duration=30.827s, table=44, n_packets=0, n_bytes=0, priority=2000,ct_state=-new+est-rpl+trk,ct_label=0/0x1,tcp,reg15=0x2,metadata=0x6,nw_src=30.1.1.112,tp_dst=5555 actions=resubmit(,45)

 cookie=0x91ef48b0, duration=4690.496s, table=44, n_packets=2, n_bytes=148, priority=2000,ct_state=+new-est+trk,tcp,reg15=0x2,metadata=0x6,nw_src=30.1.1.12,tp_dst=4444 actions=load:0x1->NXM_NX_XXREG0[97],controller(userdata=00.00.00.07.00.00.00.00.00.06.76.6d.2d.34.30.30.2d.32.2d.69.6e.2d.70.65.72.6d.69.74.34.34.34.34),resubmit(,45)

 cookie=0x4f257b76, duration=30.827s, table=44, n_packets=0, n_bytes=0, priority=2000,ct_state=+new-est+trk,tcp,reg15=0x2,metadata=0x6,nw_src=30.1.1.111,tp_dst=5555 actions=load:0x1->NXM_NX_XXREG0[97],resubmit(,45)
 cookie=0x4f257b76, duration=30.827s, table=44, n_packets=0, n_bytes=0, priority=2000,ct_state=+new-est+trk,tcp,reg15=0x2,metadata=0x6,nw_src=30.1.1.112,tp_dst=5555 actions=load:0x1->NXM_NX_XXREG0[97],resubmit(,45)
#### 4、默认acl，执行： 提交ct 表项；打日志；默认行为（drop）；设置ct_label=1
###    另外 先建ct，再删除 permit的rule表项的情况下，也走到这里，drop并设置流表的ct_label
 cookie=0x6a697aa1, duration=4690.496s, table=44, n_packets=0, n_bytes=0, priority=1000,ct_state=+est+trk,ct_label=0/0x1,ip,reg15=0x2,metadata=0x6 actions=ct(commit,zone=NXM_NX_REG13[0..15],exec(load:0x1->NXM_NX_CT_LABEL[0])),controller(userdata=00.00.00.07.00.00.00.00.01.06.76.6d.2d.34.30.30.2d.32.2d.69.6e.2d.64.65.66.61.75.6c.74)
###  到了这里，还没有ct.est 或 匹配有drop 标记的报文，打日志 + drop（没有resubmit）
###  先建ct，再配置rule，会打 ct_label，走到这里。
 cookie=0xd4099538, duration=4690.496s, table=44, n_packets=0, n_bytes=0, priority=1000,ct_state=+est+trk,ct_label=0x1/0x1,ip,reg15=0x2,metadata=0x6 actions=controller(userdata=00.00.00.07.00.00.00.00.01.06.76.6d.2d.34.30.30.2d.32.2d.69.6e.2d.64.65.66.61.75.6c.74)
### 先配置 rule，再有流量的情况下会走到这里，-est+trk，drop
 cookie=0xd4099538, duration=4690.496s, table=44, n_packets=24, n_bytes=1776, priority=1000,ct_state=-est+trk,ip,reg15=0x2,metadata=0x6 actions=controller(userdata=00.00.00.07.00.00.00.00.01.06.76.6d.2d.34.30.30.2d.32.2d.69.6e.2d.64.65.66.61.75.6c.74)
####
 cookie=0xf90599e3, duration=4690.496s, table=44, n_packets=0, n_bytes=0, priority=1,ct_state=-est+trk,ip,metadata=0x6 actions=load:0x1->NXM_NX_XXREG0[97],resubmit(,45)
 cookie=0xf90599e3, duration=4690.495s, table=44, n_packets=0, n_bytes=0, priority=1,ct_state=+est+trk,ct_label=0x1/0x1,ip,metadata=0x6 actions=load:0x1->NXM_NX_XXREG0[97],resubmit(,45)
```

# LoadBalancer

    ovn-nbctl lb-add lb_30_99 192.168.77.99:99 "30.1.1.11:9999,30.1.1.12:9999" tcp
    ovn-nbctl lr-lb-add vpc-router lb_30_99


    #### 1、 到LB VIP的 arp 代答。和OVN L3 GW一样，IP地址是不存在的，必要的协议应答全部需要流表实现
     cookie=0xdd30fe99, duration=105.234s, table=11, n_packets=1, n_bytes=42, priority=90,arp,reg14=0x3,metadata=0x2,arp_tpa=192.168.77.99,arp_op=1 actions=move:NXM_OF_ETH_SRC[]->NXM_OF_ETH_DST[],move:NXM_NX_XXREG0[64..111]->NXM_OF_ETH_SRC[],load:0x2->NXM_OF_ARP_OP[],move:NXM_NX_ARP_SHA[]->NXM_NX_ARP_THA[],move:NXM_NX_XXREG0[64..111]->NXM_NX_ARP_SHA[],push:NXM_OF_ARP_SPA[],push:NXM_OF_ARP_TPA[],pop:NXM_OF_ARP_SPA[],pop:NXM_OF_ARP_TPA[],move:NXM_NX_REG14[]->NXM_NX_REG15[],load:0x1->NXM_NX_REG10[0],resubmit(,37)
     cookie=0xc780c41a, duration=105.234s, table=11, n_packets=0, n_bytes=0, priority=90,arp,reg14=0x1,metadata=0x2,arp_tpa=192.168.77.99,arp_op=1 actions=move:NXM_OF_ETH_SRC[]->NXM_OF_ETH_DST[],move:NXM_NX_XXREG0[64..111]->NXM_OF_ETH_SRC[],load:0x2->NXM_OF_ARP_OP[],move:NXM_NX_ARP_SHA[]->NXM_NX_ARP_THA[],move:NXM_NX_XXREG0[64..111]->NXM_NX_ARP_SHA[],push:NXM_OF_ARP_SPA[],push:NXM_OF_ARP_TPA[],pop:NXM_OF_ARP_SPA[],pop:NXM_OF_ARP_TPA[],move:NXM_NX_REG14[]->NXM_NX_REG15[],load:0x1->NXM_NX_REG10[0],resubmit(,37)
     cookie=0x6ad74317, duration=105.234s, table=11, n_packets=0, n_bytes=0, priority=90,arp,reg14=0x2,metadata=0x2,arp_tpa=192.168.77.99,arp_op=1 actions=move:NXM_OF_ETH_SRC[]->NXM_OF_ETH_DST[],move:NXM_NX_XXREG0[64..111]->NXM_OF_ETH_SRC[],load:0x2->NXM_OF_ARP_OP[],move:NXM_NX_ARP_SHA[]->NXM_NX_ARP_THA[],move:NXM_NX_XXREG0[64..111]->NXM_NX_ARP_SHA[],push:NXM_OF_ARP_SPA[],push:NXM_OF_ARP_TPA[],pop:NXM_OF_ARP_SPA[],pop:NXM_OF_ARP_TPA[],move:NXM_NX_REG14[]->NXM_NX_REG15[],load:0x1->NXM_NX_REG10[0],resubmit(,37)

    ###
     cookie=0xd9e2d9dc, duration=105.234s, table=12, n_packets=8, n_bytes=550, priority=100,ip,metadata=0x2,nw_dst=192.168.77.99 actions=ct(table=13,zone=NXM_NX_REG11[0..15])

    #### 2、入向流表，非首包使用ct封包
     cookie=0x562598f9, duration=105.234s, table=14, n_packets=6, n_bytes=402, priority=120,ct_state=+est+trk,tcp,metadata=0x2,nw_dst=192.168.77.99,tp_dst=99 actions=ct(table=15,zone=NXM_NX_REG11[0..15],nat)
    ####   入向流表，首包，使用了group action做LB调度，见下面group流表： select类型的group，根据hash算法只能选择一条 bucket，并执行bucket的actions，做dnat
     cookie=0x664f0a36, duration=105.234s, table=14, n_packets=2, n_bytes=148, priority=120,ct_state=+new+trk,tcp,metadata=0x2,nw_dst=192.168.77.99,tp_dst=99 actions=group:1
    #### 3、反向流表，通过 ct nat 封装
     cookie=0x3fcb7618, duration=105.235s, table=40, n_packets=1, n_bytes=54, priority=120,tcp,reg15=0x3,metadata=0x2,nw_src=30.1.1.12,tp_src=9999 actions=ct(table=41,zone=NXM_NX_REG11[0..15],nat)
     cookie=0x3fcb7618, duration=105.235s, table=40, n_packets=5, n_bytes=339, priority=120,tcp,reg15=0x3,metadata=0x2,nw_src=30.1.1.11,tp_src=9999 actions=ct(table=41,zone=NXM_NX_REG11[0..15],nat)


     openflow group流表，ovs-ofctl dump-groups br-int 显示
     #### select类型的group，hash类型为dp_hash，为默认值，根据hash算法只能选择一条 bucket，并执行bucket的actions，做dnat
    group_id=1,type=select,selection_method=dp_hash,bucket=bucket_id:0,weight:100,actions=ct(commit,table=15,zone=NXM_NX_REG11[0..15],nat(dst=30.1.1.11:9999),exec(load:0x1->NXM_NX_CT_LABEL[1])),bucket=bucket_id:1,weight:100,actions=ct(commit,table=15,zone=NXM_NX_REG11[0..15],nat(dst=30.1.1.12:9999),exec(load:0x1->NXM_NX_CT_LABEL[1]))

    uuid=`ovn-nbctl --bare --columns _uuid find load_balancer name=lb_30_99`
    ## 配置health check ，每个后端对应的检测目的port和src ip
    ovn-nbctl --wait=sb set load_balancer $uuid ip_port_mappings:30.1.1.11=sw-300-port-vm1:30.1.1.111
    ovn-nbctl --wait=sb set load_balancer $uuid ip_port_mappings:30.1.1.12=sw-300-port-vm2:30.1.1.112
    uuid_hc=`ovn-nbctl --id=@hc create Load_Balancer_Health_Check vip="192.168.77.99\:99" -- add Load_Balancer $uuid health_check @hc`
    ovn-nbctl set Load_Balancer_Health_Check $uuid_hc options:interval=5 options:timeout=20 options:success_count=3 options:failure_count=3

    流表变化
    #### 增加了 health check的 src ip的 arp 代答
     cookie=0xa8cab54f, duration=237.147s, table=24, n_packets=6, n_bytes=252, priority=110,arp,metadata=0x1,arp_tpa=30.1.1.111,arp_op=1 actions=move:NXM_OF_ETH_SRC[]->NXM_OF_ETH_DST[],mod_dl_src:3e:f5:0e:5f:8b:e8,load:0x2->NXM_OF_ARP_OP[],move:NXM_NX_ARP_SHA[]->NXM_NX_ARP_THA[],load:0x3ef50e5f8be8->NXM_NX_ARP_SHA[],move:NXM_OF_ARP_SPA[]->NXM_OF_ARP_TPA[],load:0x1e01016f->NXM_OF_ARP_SPA[],move:NXM_NX_REG14[]->NXM_NX_REG15[],load:0x1->NXM_NX_REG10[0],resubmit(,37)
     cookie=0x1b34bc, duration=233.012s, table=24, n_packets=5, n_bytes=210, priority=110,arp,metadata=0x1,arp_tpa=30.1.1.112,arp_op=1 actions=move:NXM_OF_ETH_SRC[]->NXM_OF_ETH_DST[],mod_dl_src:3e:f5:0e:5f:8b:e8,load:0x2->NXM_OF_ARP_OP[],move:NXM_NX_ARP_SHA[]->NXM_NX_ARP_THA[],load:0x3ef50e5f8be8->NXM_NX_ARP_SHA[],move:NXM_OF_ARP_SPA[]->NXM_OF_ARP_TPA[],load:0x1e010170->NXM_OF_ARP_SPA[],move:NXM_NX_REG14[]->NXM_NX_REG15[],load:0x1->NXM_NX_REG10[0],resubmit(,37)


    内核流表
    [root@centrial ~]# ovs-dpctl dump-flows  ovs-system
    2021-12-28T03:37:34Z|00001|netdev_linux|INFO|ioctl(SIOCGIFINDEX) on sw-300-port-vm2 device failed: No such device
    2021-12-28T03:37:34Z|00002|netdev_linux|INFO|ioctl(SIOCGIFINDEX) on sw-300-port-vm1 device failed: No such device
    recirc_id(0x78),in_port(3),ct_state(-new+est+trk),eth(src=02:d4:1d:8c:ff:01,dst=ba:1e:14:c9:ca:dd),eth_type(0x0800),ipv4(frag=no), packets:11, bytes:918, used:2.376s, flags:FP., actions:ct_clear,8
    recirc_id(0x76),in_port(8),ct_state(-new+est+trk),eth(src=ba:1e:14:c9:ca:dd,dst=02:d4:1d:8c:ff:01),eth_type(0x0800),ipv4(dst=30.1.1.11,ttl=64,frag=no), packets:10, bytes:660, used:4.692s, flags:F., actions:ct_clear,set(eth(src=02:d4:1d:8c:30:01,dst=aa:10:dd:1b:30:01)),set(ipv4(ttl=63)),5
    recirc_id(0),in_port(5),ct_state(-new-est-trk),eth(src=00:00:00:00:00:00/01:00:00:00:00:00,dst=3e:f5:0e:5f:8b:e8),eth_type(0x0800),ipv4(frag=no), packets:1274, bytes:72172, used:2.754s, flags:SR., actions:userspace(pid=4294963083,controller(reason=1,dont_send=0,continuation=0,recirc_id=18,rule_cookie=0x75c46944,controller_id=0,max_len=65535))
    recirc_id(0x76),in_port(8),ct_state(-new+est+trk),eth(src=ba:1e:14:c9:ca:dd,dst=02:d4:1d:8c:ff:01),eth_type(0x0800),ipv4(dst=30.1.1.12,ttl=64,frag=no), packets:9, bytes:594, used:2.376s, flags:F., actions:ct_clear,set(eth(src=02:d4:1d:8c:30:01,dst=aa:10:dd:1b:30:02)),set(ipv4(ttl=63)),3
    ### 1、入向，80.1.1.1 的源，ct创建前流表
    recirc_id(0x75),in_port(8),ct_state(+new+trk),eth(),eth_type(0x0800),ipv4(src=80.1.1.1,dst=192.168.77.99,proto=6,frag=no),tcp(dst=99), packets:0, bytes:0, used:never, actions:ct(commit,zone=8,label=0x2/0x2,nat(dst=30.1.1.11:9999)),recirc(0x76)
    recirc_id(0x77),in_port(5),ct_state(-new+est+trk),eth(src=02:d4:1d:8c:ff:01,dst=ba:1e:14:c9:ca:dd),eth_type(0x0800),ipv4(frag=no), packets:11, bytes:918, used:4.692s, flags:FP., actions:ct_clear,8
    recirc_id(0x76),in_port(8),ct_state(+new-est+trk),eth(src=ba:1e:14:c9:ca:dd,dst=02:d4:1d:8c:ff:01),eth_type(0x0800),ipv4(dst=30.1.1.11,ttl=64,frag=no), packets:0, bytes:0, used:never, actions:ct_clear,set(eth(src=02:d4:1d:8c:30:01,dst=aa:10:dd:1b:30:01)),set(ipv4(ttl=63)),5
    recirc_id(0),in_port(8),ct_state(-new-est-trk),eth(src=ba:1e:14:c9:ca:dd,dst=02:d4:1d:8c:ff:01),eth_type(0x0800),ipv4(src=64.0.0.0/224.0.0.0,dst=192.168.77.99,proto=6,ttl=64,frag=no), packets:24, bytes:1752, used:2.376s, flags:SFP., actions:ct_clear,ct(zone=8),recirc(0x75)
    recirc_id(0),in_port(3),ct_state(-new-est-trk),eth(src=00:00:00:00:00:00/01:00:00:00:00:00,dst=3e:f5:0e:5f:8b:e8),eth_type(0x0800),ipv4(frag=no), packets:1274, bytes:72292, used:2.754s, flags:SR., actions:userspace(pid=4294963085,controller(reason=1,dont_send=0,continuation=0,recirc_id=17,rule_cookie=0x75c46944,controller_id=0,max_len=65535))
    recirc_id(0),in_port(5),ct_state(-new-est-trk),eth(src=aa:10:dd:1b:30:01,dst=02:d4:1d:8c:30:01),eth_type(0x0800),ipv4(src=30.1.1.11,dst=64.0.0.0/224.0.0.0,proto=6,ttl=64,frag=no),tcp(src=9999), packets:11, bytes:918, used:4.692s, flags:FP., actions:ct_clear,set(eth(src=02:d4:1d:8c:ff:01,dst=ba:1e:14:c9:ca:dd)),set(ipv4(ttl=63)),ct(zone=8,nat),recirc(0x77)
    recirc_id(0),in_port(3),ct_state(-new-est-trk),eth(src=aa:10:dd:1b:30:02,dst=02:d4:1d:8c:30:01),eth_type(0x0800),ipv4(src=30.1.1.12,dst=64.0.0.0/224.0.0.0,proto=6,ttl=64,frag=no),tcp(src=9999), packets:11, bytes:918, used:2.376s, flags:FP., actions:ct_clear,set(eth(src=02:d4:1d:8c:ff:01,dst=ba:1e:14:c9:ca:dd)),set(ipv4(ttl=63)),ct(zone=8,nat),recirc(0x78)
    ### 3、入向，ct建成后流表
    recirc_id(0x75),in_port(8),ct_state(-new+est+trk),eth(),eth_type(0x0800),ipv4(dst=192.168.77.99,proto=6,frag=no),tcp(dst=99), packets:22, bytes:1612, used:2.376s, flags:FP., actions:ct(zone=8,nat),recirc(0x76)
    ### 2、入向，77.1.1.1 的源，ct创建前流表
    recirc_id(0x75),in_port(8),ct_state(+new+trk),eth(),eth_type(0x0800),ipv4(src=77.1.1.1,dst=192.168.77.99,proto=6,frag=no),tcp(dst=99), packets:0, bytes:0, used:never, actions:ct(commit,zone=8,label=0x2/0x2,nat(dst=30.1.1.12:9999)),recirc(0x76)
    recirc_id(0x76),in_port(8),ct_state(+new-est+trk),eth(src=ba:1e:14:c9:ca:dd,dst=02:d4:1d:8c:ff:01),eth_type(0x0800),ipv4(dst=30.1.1.12,ttl=64,frag=no), packets:0, bytes:0, used:never, actions:ct_clear,set(eth(src=02:d4:1d:8c:30:01,dst=aa:10:dd:1b:30:02)),set(ipv4(ttl=63)),3

https://www.jianshu.com/p/45fd05700682

