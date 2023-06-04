### 流表寄存器意义

| **寄存器** | **功能** | **详解** |
| :--- | :--- | :--- |
| metadata | 作为vni使用 | 是ovn的Logical Datapath Field，命令ovn-sbctl list Datapath\_Binding查看tunnel\_key，封装到geneve或者stt中 |
| reg14 | 记录逻辑入端口 | 是ovn的Logical InputPort Field，命令ovn-sbctl list Port\_Binding查看tunnel\_key，封装到geneve或者stt中 |
| reg15 | 记录逻辑出端口 | 是ovn的Logical OutputPort Field，命令ovn-sbctl list Port\_Binding查看tunnel\_key，封装到geneve或者stt中 |
| reg13 | 逻辑端口的conntrack zone | chassis内部有用，出了chassis无用 |
| reg12 | SNAT的conntrack zone | 也是chassis内部使用 |
| reg11 | DNAT的conntrack zone | 也是chassis内部使用 |
| reg10 | 逻辑流表标志 | 可能是逻辑流表中的flags.loopback之类的标志 |

### OVS流表

### 流表分析

**table 0主要工作如下：**

l  完成物理到逻辑的翻译，将逻辑信息，比如上面提到的信息记录到寄存器中。

l  VM中的容器的报文用VLAN进行区分

l  别的chassis过来的报文，根据入端口和tunnel\_id进行区分，然后获取出端口，这个在封装的时候已经有了

**table 16-31主要是将逻辑流表ingress pipeline 0-15的操作部分转换为openflow流表，主要工作如下：**

l  每个逻辑流表会映射一个或者多个openflow流表，通常报文只是匹配其中一条流表。

l  ovn-controller使用逻辑流表的UUID的前32位作为openflow流表的cookie值。查看逻辑流表的UUID使用ovn-sbctl list Logical\_Flow，对应上面cookie的逻辑流表的UUID的信息在这里。

l  一些逻辑流表可以映射到ovs的”conjunctive match”扩展名\(参见这里\)，这时候因为一条openflow流表对应了多条逻辑流表，所以cookie为0。这里的”conjunctive match”表示一个集合的匹配，比如tcp\_src ∈ {80, 443, 8080} and tcp\_dst ∈ {80, 443, 8080}。

l  一些逻辑流表可能不会转换成openflow流表，如果交换机上虚拟接口没有添加到ovs中，添加命令ovs-vsctl set Interface veth2\_b external\_ids:iface-id=ls2-vm4，那么相应的openflow流表将不会生成。

l  最后就是有一些逻辑流表和openflow流表很明显的对应操作关系，我们列一下

l  next对应resubmit

l  field = constant对应set\_field

l  output，将报文resubmit到表32，如果逻辑流表有多个output操作，那么每个都要resubmit到表32。

l  get\_arp\(P, A\)和get\_nd\(P, A\)，通过讲参数存储在openflow字段中\(上面例子中存储在NXM\_NX\_REG0，流表cookie=0x5dbc664\)，然后resubmit到表66，然后ovn-controller从MAC\_Binding表生成流填充，如果表66中有匹配项，其action将绑定的MAC存储在目的MAC地址字段中

l  put\_arp\(P, A, E\)和put\_nd\(P, A, E\)讲参数存储到openflow的字段中\(字段太多，查看上面流表cookie=0x92af5d1c\)，然后更新MAC\_Binding表中。

**table 32-47主要是将逻辑流表ingress pipeline的output action转换为openflow流表。以下详细介绍下：**

**表32主要是处理到其他宿主机中虚拟机的报文，讲VNI设置到metadata，然后resubmit到表33**

**表33主要是将报文resubmit到表34，对于多个逻辑output端口的时候，需要改为每个逻辑端口P，然后resubmit到表34**

**表34检查报文的逻辑ingress和egress的端口是否一致，一致则丢弃。剩下的resubmit到表48**

**table 48-63主要是讲逻辑流表的egress pipeline部分转换成openflow流表，这块属于报文发送之前的最后验证，最终resubmit到表64，最终没有执行output的报文将被丢弃。**

**table 64貌似和loopback有关，修改逻辑入端口。**

**table 65逻辑到物理的转换，和表0相反，主要是将找到逻辑端口对应的物理端口，然后发送，如果虚拟机中还有容器的话，需要添加vlan头。**

**table 66主要是对应MAC\_Binding中的数据，来修改目的IP对应的目的MAC，功能类似arp。**

```
# ovs-ofctl dump-flows br-int 

//cookie没有值表示不是直接从逻辑流表转换而来的

//两个虚拟机进来的报文进行一些寄存器的操作，这个不是根据逻辑流表来的，但是和逻辑拓扑还是有关系的，具体这些寄存器的意义和获取我们下面介绍

 cookie=0x0, table=0, priority=100,in_port=4 actions=load:0x1->NXM_NX_REG13[],load:0x6->NXM_NX_REG11[],load:0x8->NXM_NX_REG12[],load:0x3->OXM_OF_METADATA[],load:0x2->NXM_NX_REG14[],resubmit(,16)

 cookie=0x0, table=0, priority=100,in_port=3 actions=load:0x2->NXM_NX_REG13[],load:0x7->NXM_NX_REG11[],load:0x5->NXM_NX_REG12[],load:0x2->OXM_OF_METADATA[],load:0x2->NXM_NX_REG14[],resubmit(,16)

 //表示从其他宿主机发送过来的报文应该如何处理，这里的tun_id分别表示从两个逻辑交换中的哪一个发送过来的

 cookie=0x0, table=0, priority=100,tun_id=0x3,in_port=7 actions=move:NXM_NX_TUN_ID[0..23]->OXM_OF_METADATA[0..23],load:0x3->NXM_NX_REG14[0..14],load:0x1->NXM_NX_REG10[1],resubmit(,16)

 cookie=0x0, table=0, priority=100,tun_id=0x2,in_port=7 actions=move:NXM_NX_TUN_ID[0..23]->OXM_OF_METADATA[0..23],load:0x3->NXM_NX_REG14[0..14],load:0x1->NXM_NX_REG10[1],resubmit(,16)



 //一些我们不关注的流表主要是一些错误报文的丢弃操作，相关流表已经删除了

 //以下metadata不是1表示从逻辑交换发过来的报文怎么处理，前面的reg14表示从哪个逻辑端口发送过来的

 cookie=0xa7c014e8, table=16, priority=50,reg14=0x2,metadata=0x3,dl_src=52:54:00:c1:68:71 actions=resubmit(,17)

 cookie=0x3ed26758, table=16, priority=50,reg14=0x2,metadata=0x2,dl_src=52:54:00:c1:68:70 actions=resubmit(,17)

 cookie=0x11dd5c04, table=16, priority=50,reg14=0x3,metadata=0x2,dl_src=52:54:00:c1:68:72 actions=resubmit(,17)

 cookie=0x6126e3c1, table=16, priority=50,reg14=0x3,metadata=0x3,dl_src=52:54:00:c1:68:73 actions=resubmit(,17)

 cookie=0x75e7ab7b, table=16, priority=50,reg14=0x1,metadata=0x2 actions=resubmit(,17)

 cookie=0x8c78254f, table=16, priority=50,reg14=0x1,metadata=0x3 actions=resubmit(,17)

 //以下metadata为1表示从逻辑路由过来的报文，需要进行怎样的操作

 cookie=0xd9caf1fd, table=16, priority=50,reg14=0x1,metadata=0x1,dl_dst=01:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,17)

 cookie=0xeac605df, table=16, priority=50,reg14=0x2,metadata=0x1,dl_dst=01:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,17)

 cookie=0x819b5118, table=16, priority=50,reg14=0x1,metadata=0x1,dl_dst=52:54:00:c1:68:50 actions=resubmit(,17)

 cookie=0xbe725a2b, table=16, priority=50,reg14=0x2,metadata=0x1,dl_dst=52:54:00:c1:68:60 actions=resubmit(,17)



 //arp代答的流表

 cookie=0xf4ca156, table=17, priority=90,arp,reg14=0x2,metadata=0x1,arp_tpa=192.168.2.1,arp_op=1 actions=move:NXM_OF_ETH_SRC[]->NXM_OF_ETH_DST[],mod_dl_src:52:54:00:c1:68:60,load:0x2->NXM_OF_ARP_OP[],move:NXM_NX_ARP_SHA[]->NXM_NX_ARP_THA[],load:0x525400c16860->NXM_NX_ARP_SHA[],move:NXM_OF_ARP_SPA[]->NXM_OF_ARP_TPA[],load:0xc0a80201->NXM_OF_ARP_SPA[],load:0x2->NXM_NX_REG15[],load:0x1->NXM_NX_REG10[0],resubmit(,32)

 cookie=0xb5d8c2e4, table=17, priority=90,arp,reg14=0x1,metadata=0x1,arp_tpa=192.168.1.1,arp_op=1 actions=move:NXM_OF_ETH_SRC[]->NXM_OF_ETH_DST[],mod_dl_src:52:54:00:c1:68:50,load:0x2->NXM_OF_ARP_OP[],move:NXM_NX_ARP_SHA[]->NXM_NX_ARP_THA[],load:0x525400c16850->NXM_NX_ARP_SHA[],move:NXM_OF_ARP_SPA[]->NXM_OF_ARP_TPA[],load:0xc0a80101->NXM_OF_ARP_SPA[],load:0x1->NXM_NX_REG15[],load:0x1->NXM_NX_REG10[0],resubmit(,32)

 //arp回复报文的信息存入MAC_Binding

 cookie=0x92af5d1c, table=17, priority=90,arp,metadata=0x1,arp_op=2 actions=push:NXM_NX_REG0[],push:NXM_OF_ETH_SRC[],push:NXM_NX_ARP_SHA[],push:NXM_OF_ARP_SPA[],pop:NXM_NX_REG0[],pop:NXM_OF_ETH_SRC[],controller(userdata=00.00.00.01.00.00.00.00),pop:NXM_OF_ETH_SRC[],pop:NXM_NX_REG0[]

 //icmp代答

 cookie=0x815a3063, table=17, priority=90,icmp,metadata=0x1,nw_dst=192.168.1.1,icmp_type=8,icmp_code=0 actions=push:NXM_OF_IP_SRC[],push:NXM_OF_IP_DST[],pop:NXM_OF_IP_SRC[],pop:NXM_OF_IP_DST[],load:0xff->NXM_NX_IP_TTL[],load:0->NXM_OF_ICMP_TYPE[],load:0x1->NXM_NX_REG10[0],resubmit(,18)

 cookie=0xf3d609b1, table=17, priority=90,icmp,metadata=0x1,nw_dst=192.168.2.1,icmp_type=8,icmp_code=0 actions=push:NXM_OF_IP_SRC[],push:NXM_OF_IP_DST[],pop:NXM_OF_IP_SRC[],pop:NXM_OF_IP_DST[],load:0xff->NXM_NX_IP_TTL[],load:0->NXM_OF_ICMP_TYPE[],load:0x1->NXM_NX_REG10[0],resubmit(,18)

//三个逻辑设备的流量继续往下走

 cookie=0x56295f89, table=17, priority=0,metadata=0x1 actions=resubmit(,18)

 cookie=0x791195e0, table=17, priority=0,metadata=0x3 actions=resubmit(,18)

 cookie=0x4b1c93d4, table=17, priority=0,metadata=0x2 actions=resubmit(,18)



 //arp通过

 cookie=0x4a80a501, table=18, priority=90,arp,reg14=0x3,metadata=0x3,dl_src=52:54:00:c1:68:73,arp_sha=52:54:00:c1:68:73 actions=resubmit(,19)

 cookie=0xc6c881ee, table=18, priority=90,arp,reg14=0x3,metadata=0x2,dl_src=52:54:00:c1:68:72,arp_sha=52:54:00:c1:68:72 actions=resubmit(,19)

 //继续

 cookie=0xb76a420f, table=18, priority=0,metadata=0x2 actions=resubmit(,19)

 cookie=0x3ecbeeec, table=18, priority=0,metadata=0x1 actions=resubmit(,19)

 cookie=0x78c16fb8, table=18, priority=0,metadata=0x3 actions=resubmit(,19)



 //继续

 cookie=0x76f9414c, table=19, priority=0,metadata=0x3 actions=resubmit(,20)

 cookie=0xff75779d, table=19, priority=0,metadata=0x2 actions=resubmit(,20)

 cookie=0xa4a71b19, table=19, priority=0,metadata=0x1 actions=resubmit(,20)



 //继续

 cookie=0x4c209f08, table=20, priority=0,metadata=0x3 actions=resubmit(,21)

 cookie=0xc99c5154, table=20, priority=0,metadata=0x1 actions=resubmit(,21)

 cookie=0xe187a6b4, table=20, priority=0,metadata=0x2 actions=resubmit(,21)



 //conntrack记录

 cookie=0x5c49d2d2, table=21, priority=100,ip,reg0=0x1/0x1,metadata=0x3 actions=ct(table=22,zone=NXM_NX_REG13[0..15])

 cookie=0x596e0c95, table=21, priority=100,ip,reg0=0x1/0x1,metadata=0x2 actions=ct(table=22,zone=NXM_NX_REG13[0..15])

 //模拟过网关时的操作

 cookie=0xaea49216, table=21, priority=49,ip,metadata=0x1,nw_dst=192.168.1.0/24 actions=dec_ttl(),move:NXM_OF_IP_DST[]->NXM_NX_XXREG0[96..127],load:0xc0a80101->NXM_NX_XXREG0[64..95],mod_dl_src:52:54:00:c1:68:50,load:0x1->NXM_NX_REG15[],load:0x1->NXM_NX_REG10[0],resubmit(,22)

 cookie=0x3ebae949, table=21, priority=49,ip,metadata=0x1,nw_dst=192.168.2.0/24 actions=dec_ttl(),move:NXM_OF_IP_DST[]->NXM_NX_XXREG0[96..127],load:0xc0a80201->NXM_NX_XXREG0[64..95],mod_dl_src:52:54:00:c1:68:60,load:0x2->NXM_NX_REG15[],load:0x1->NXM_NX_REG10[0],resubmit(,22)

 //继续

 cookie=0xe3a08e2b, table=21, priority=0,metadata=0x3 actions=resubmit(,22)

 cookie=0x80407476, table=21, priority=0,metadata=0x2 actions=resubmit(,22)



 //获取MAC_Binding表里的数据，回复arp

 cookie=0x5dbc664, table=22, priority=0,ip,metadata=0x1 actions=push:NXM_NX_REG0[],push:NXM_NX_XXREG0[96..127],pop:NXM_NX_REG0[],mod_dl_dst:00:00:00:00:00:00,resubmit(,66),pop:NXM_NX_REG0[],resubmit(,23)

 //继续

 cookie=0x66236a1, table=22, priority=0,metadata=0x2 actions=resubmit(,23)

 cookie=0xefaed143, table=22, priority=0,metadata=0x3 actions=resubmit(,23)



 //继续

 cookie=0x3998ed82, table=23, priority=0,metadata=0x1 actions=resubmit(,24)

 cookie=0xc475a7b3, table=23, priority=0,metadata=0x3 actions=resubmit(,24)

 cookie=0xacda159d, table=23, priority=0,metadata=0x2 actions=resubmit(,24)



 //????发送arp？

 cookie=0xe51fffad, table=24, priority=100,ip,metadata=0x1,dl_dst=00:00:00:00:00:00 actions=controller(userdata=00.00.00.00.00.00.00.00.00.19.00.10.80.00.06.06.ff.ff.ff.ff.ff.ff.00.00.ff.ff.00.18.00.00.23.20.00.06.00.20.00.40.00.00.00.01.de.10.00.00.20.04.ff.ff.00.18.00.00.23.20.00.06.00.20.00.60.00.00.00.01.de.10.00.00.22.04.00.19.00.10.80.00.2a.02.00.01.00.00.00.00.00.00.ff.ff.00.10.00.00.23.20.00.0e.ff.f8.20.00.00.00)

 //继续

 cookie=0xd9c9912b, table=24, priority=0,metadata=0x1 actions=resubmit(,32)

 cookie=0x9b703aff, table=24, priority=0,metadata=0x2 actions=resubmit(,25)

 cookie=0xd44f4b41, table=24, priority=0,metadata=0x3 actions=resubmit(,25)



 //conntrack lb

 cookie=0xed10c525, table=25, priority=100,ip,reg0=0x4/0x4,metadata=0x3 actions=ct(table=26,zone=NXM_NX_REG13[0..15],nat)

 cookie=0xb0869023, table=25, priority=100,ip,reg0=0x4/0x4,metadata=0x2 actions=ct(table=26,zone=NXM_NX_REG13[0..15],nat)

 //conntrack

 cookie=0xc8dfda6d, table=25, priority=100,ip,reg0=0x2/0x2,metadata=0x2 actions=ct(commit,zone=NXM_NX_REG13[0..15],exec(load:0->NXM_NX_CT_LABEL[0])),resubmit(,26)

 cookie=0xf71a37ba, table=25, priority=100,ip,reg0=0x2/0x2,metadata=0x3 actions=ct(commit,zone=NXM_NX_REG13[0..15],exec(load:0->NXM_NX_CT_LABEL[0])),resubmit(,26)

 //继续

 cookie=0x3c4b37a7, table=25, priority=0,metadata=0x2 actions=resubmit(,26)

 cookie=0x315f30b3, table=25, priority=0,metadata=0x3 actions=resubmit(,26)



 //继续

 cookie=0x4368d2e8, table=26, priority=0,metadata=0x3 actions=resubmit(,27)

 cookie=0xf906a487, table=26, priority=0,metadata=0x2 actions=resubmit(,27)

 cookie=0x1ab8df97, table=27, priority=0,metadata=0x3 actions=resubmit(,28)


 //泛洪

 cookie=0x159f7998, table=29, priority=100,metadata=0x3,dl_dst=01:00:00:00:00:00/01:00:00:00:00:00 actions=load:0xffff->NXM_NX_REG15[],resubmit(,32)

 cookie=0xcbb8e72a, table=29, priority=100,metadata=0x2,dl_dst=01:00:00:00:00:00/01:00:00:00:00:00 actions=load:0xffff->NXM_NX_REG15[],resubmit(,32)

 //出口流量

 cookie=0xc0e4e6a6, table=29, priority=50,metadata=0x2,dl_dst=52:54:00:c1:68:72 actions=load:0x3->NXM_NX_REG15[],resubmit(,32)

 cookie=0x13381c84, table=29, priority=50,metadata=0x3,dl_dst=52:54:00:c1:68:73 actions=load:0x3->NXM_NX_REG15[],resubmit(,32)

 cookie=0x23555b13, table=29, priority=50,metadata=0x2,dl_dst=52:54:00:c1:68:50 actions=load:0x1->NXM_NX_REG15[],resubmit(,32)



 //????没有flags为2的标志

 cookie=0x0, table=32, priority=150,reg10=0x2/0x2 actions=resubmit(,33)

 //到逻辑路由的流量

 cookie=0x0, table=32, priority=100,reg15=0xffff,metadata=0x3 actions=load:0x1->NXM_NX_REG15[],resubmit(,34),load:0xffff->NXM_NX_REG15[],load:0x3->NXM_NX_TUN_ID[0..23],output:7,resubmit(,33)

 cookie=0x0, table=32, priority=100,reg15=0xffff,metadata=0x2 actions=load:0x1->NXM_NX_REG15[],resubmit(,34),load:0xffff->NXM_NX_REG15[],load:0x2->NXM_NX_TUN_ID[0..23],output:7,resubmit(,33)

 //到逻辑交换的流量

 cookie=0x0, table=32, priority=100,reg15=0x3,metadata=0x2 actions=load:0x2->NXM_NX_TUN_ID[0..23],output:7

 cookie=0x0, table=32, priority=100,reg15=0x3,metadata=0x3 actions=load:0x3->NXM_NX_TUN_ID[0..23],output:7

 //继续

 cookie=0x0, table=32, priority=0 actions=resubmit(,33)



 //????到网络节点需要NAT的流量，可是我们没有相应的配置

 cookie=0x0, table=33, priority=100,reg15=0x1,metadata=0x3 actions=load:0x6->NXM_NX_REG11[],load:0x8->NXM_NX_REG12[],resubmit(,34)

 cookie=0x0, table=33, priority=100,reg15=0x2,metadata=0x1 actions=load:0x3->NXM_NX_REG11[],load:0x4->NXM_NX_REG12[],resubmit(,34)

 cookie=0x0, table=33, priority=100,reg15=0x2,metadata=0x2 actions=load:0x2->NXM_NX_REG13[],load:0x7->NXM_NX_REG11[],load:0x5->NXM_NX_REG12[],resubmit(,34)

 cookie=0x0, table=33, priority=100,reg15=0x1,metadata=0x2 actions=load:0x7->NXM_NX_REG11[],load:0x5->NXM_NX_REG12[],resubmit(,34)

 cookie=0x0, table=33, priority=100,reg15=0x1,metadata=0x1 actions=load:0x3->NXM_NX_REG11[],load:0x4->NXM_NX_REG12[],resubmit(,34)

 cookie=0x0, table=33, priority=100,reg15=0x2,metadata=0x3 actions=load:0x1->NXM_NX_REG13[],load:0x6->NXM_NX_REG11[],load:0x8->NXM_NX_REG12[],resubmit(,34)

 //继续

 cookie=0x0, table=33, priority=100,reg15=0xffff,metadata=0x2 actions=load:0x2->NXM_NX_REG13[],load:0x2->NXM_NX_REG15[],resubmit(,34),load:0xffff->NXM_NX_REG15[]

 cookie=0x0, table=33, priority=100,reg15=0xffff,metadata=0x3 actions=load:0x1->NXM_NX_REG13[],load:0x2->NXM_NX_REG15[],resubmit(,34),load:0xffff->NXM_NX_REG15[]



 //继续

 cookie=0x0, table=34, priority=0 actions=load:0->NXM_NX_REG0[],load:0->NXM_NX_REG1[],load:0->NXM_NX_REG2[],load:0->NXM_NX_REG3[],load:0->NXM_NX_REG4[],load:0->NXM_NX_REG5[],load:0->NXM_NX_REG6[],load:0->NXM_NX_REG7[],load:0->NXM_NX_REG8[],load:0->NXM_NX_REG9[],resubmit(,48)



 //继续

 cookie=0x38579acc, table=48, priority=0,metadata=0x1 actions=resubmit(,49)

 cookie=0x402567e, table=48, priority=0,metadata=0x3 actions=resubmit(,49)

 cookie=0x7e6e093d, table=48, priority=0,metadata=0x2 actions=resubmit(,49)



 //继续

 cookie=0xbce65dae, table=49, priority=0,metadata=0x2 actions=resubmit(,50)

 cookie=0xf6e47c0e, table=49, priority=0,metadata=0x1 actions=resubmit(,50)

 cookie=0xa630e910, table=49, priority=0,metadata=0x3 actions=resubmit(,50)



 //conntrack

 cookie=0xe6e35197, table=50, priority=100,ipv6,reg0=0x1/0x1,metadata=0x3 actions=ct(table=51,zone=NXM_NX_REG13[0..15])

 cookie=0xa7a5e5f3, table=50, priority=100,ipv6,reg0=0x1/0x1,metadata=0x2 actions=ct(table=51,zone=NXM_NX_REG13[0..15])

 cookie=0xa7a5e5f3, table=50, priority=100,ip,reg0=0x1/0x1,metadata=0x2 actions=ct(table=51,zone=NXM_NX_REG13[0..15])

 cookie=0xe6e35197, table=50, priority=100,ip,reg0=0x1/0x1,metadata=0x3 actions=ct(table=51,zone=NXM_NX_REG13[0..15])

 //继续

 cookie=0x4e268323, table=50, priority=0,metadata=0x1 actions=resubmit(,51)

 cookie=0x2e28bd0c, table=50, priority=0,metadata=0x2 actions=resubmit(,51)

 cookie=0x7cca0b71, table=50, priority=0,metadata=0x3 actions=resubmit(,51)



 //需要输出到逻辑路由的流量

 cookie=0x1c84ef4, table=51, priority=100,reg15=0x2,metadata=0x1 actions=resubmit(,64)

 cookie=0x83ce9e62, table=51, priority=100,reg15=0x1,metadata=0x1 actions=resubmit(,64)

 //继续

 cookie=0x51c9cccf, table=51, priority=0,metadata=0x2 actions=resubmit(,52)

 cookie=0x7778d918, table=51, priority=0,metadata=0x3 actions=resubmit(,52)



 //继续

 cookie=0xa9ae4aaa, table=52, priority=0,metadata=0x2 actions=resubmit(,53)

 cookie=0xe190604a, table=52, priority=0,metadata=0x3 actions=resubmit(,53)

 cookie=0x934c95d9, table=53, priority=0,metadata=0x3 actions=resubmit(,54)

 cookie=0x828e0c10, table=53, priority=0,metadata=0x2 actions=resubmit(,54)



 //conntrack lb

 cookie=0xb1d05c18, table=54, priority=100,ip,reg0=0x4/0x4,metadata=0x3 actions=ct(table=55,zone=NXM_NX_REG13[0..15],nat)

 cookie=0x4b8234d9, table=54, priority=100,ip,reg0=0x4/0x4,metadata=0x2 actions=ct(table=55,zone=NXM_NX_REG13[0..15],nat)

 //conntrack

 cookie=0x6027420b, table=54, priority=100,ip,reg0=0x2/0x2,metadata=0x3 actions=ct(commit,zone=NXM_NX_REG13[0..15],exec(load:0->NXM_NX_CT_LABEL[0])),resubmit(,55)

 cookie=0x76bd97bd, table=54, priority=100,ip,reg0=0x2/0x2,metadata=0x2 actions=ct(commit,zone=NXM_NX_REG13[0..15],exec(load:0->NXM_NX_CT_LABEL[0])),resubmit(,55)

 //继续

 cookie=0x390ebf5f, table=54, priority=0,metadata=0x2 actions=resubmit(,55)

 cookie=0x6537ab93, table=54, priority=0,metadata=0x3 actions=resubmit(,55)

 cookie=0x13159847, table=55, priority=0,metadata=0x3 actions=resubmit(,56)

 cookie=0x439f6726, table=55, priority=0,metadata=0x2 actions=resubmit(,56)



 //多播流量

 cookie=0xb5641b45, table=56, priority=100,metadata=0x2,dl_dst=01:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,64)

 cookie=0x7b1296c4, table=56, priority=100,metadata=0x3,dl_dst=01:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,64)

 //到某个虚拟机的流量

 cookie=0xcfbbf747, table=56, priority=50,reg15=0x3,metadata=0x2,dl_dst=52:54:00:c1:68:72 actions=resubmit(,64)

 cookie=0xd39cd78f, table=56, priority=50,reg15=0x3,metadata=0x3,dl_dst=52:54:00:c1:68:73 actions=resubmit(,64)

 cookie=0x46f7518d, table=56, priority=50,reg15=0x2,metadata=0x3,dl_dst=52:54:00:c1:68:71 actions=resubmit(,64)

 cookie=0x10683faf, table=56, priority=50,reg15=0x2,metadata=0x2,dl_dst=52:54:00:c1:68:70 actions=resubmit(,64)

 //继续

 cookie=0xdf1a835, table=56, priority=50,reg15=0x1,metadata=0x3 actions=resubmit(,64)

 cookie=0x69d25440, table=56, priority=50,reg15=0x1,metadata=0x2 actions=resubmit(,64)



 //修改入端口，为重新循环做准备

 cookie=0x0, table=64, priority=100,reg10=0x1/0x1,reg15=0x1,metadata=0x1 actions=push:NXM_OF_IN_PORT[],load:0->NXM_OF_IN_PORT[],resubmit(,65),pop:NXM_OF_IN_PORT[]

 cookie=0x0, table=64, priority=100,reg10=0x1/0x1,reg15=0x2,metadata=0x3 actions=push:NXM_OF_IN_PORT[],load:0->NXM_OF_IN_PORT[],resubmit(,65),pop:NXM_OF_IN_PORT[]


 cookie=0x0, table=64, priority=0 actions=resubmit(,65)



 //将报文重新resubmit到表16，表示过完一个逻辑网元，需要进入下一个逻辑网元了

 cookie=0x0, table=65, priority=100,reg15=0x2,metadata=0x1 actions=clone(ct_clear,load:0->NXM_NX_REG11[],load:0->NXM_NX_REG12[],load:0->NXM_NX_REG13[],load:0x6->NXM_NX_REG11[],load:0x8->NXM_NX_REG12[],load:0x3->OXM_OF_METADATA[],load:0x1->NXM_NX_REG14[],load:0->NXM_NX_REG10[],load:0->NXM_NX_REG15[],load:0->NXM_NX_REG0[],load:0->NXM_NX_REG1[],load:0->NXM_NX_REG2[],load:0->NXM_NX_REG3[],load:0->NXM_NX_REG4[],load:0->NXM_NX_REG5[],load:0->NXM_NX_REG6[],load:0->NXM_NX_REG7[],load:0->NXM_NX_REG8[],load:0->NXM_NX_REG9[],load:0->NXM_OF_IN_PORT[],resubmit(,16))

 cookie=0x0, table=65, priority=100,reg15=0x1,metadata=0x2 actions=clone(ct_clear,load:0->NXM_NX_REG11[],load:0->NXM_NX_REG12[],load:0->NXM_NX_REG13[],load:0x3->NXM_NX_REG11[],load:0x4->NXM_NX_REG12[],load:0x1->OXM_OF_METADATA[],load:0x1->NXM_NX_REG14[],load:0->NXM_NX_REG10[],load:0->NXM_NX_REG15[],load:0->NXM_NX_REG0[],load:0->NXM_NX_REG1[],load:0->NXM_NX_REG2[],load:0->NXM_NX_REG3[],load:0->NXM_NX_REG4[],load:0->NXM_NX_REG5[],load:0->NXM_NX_REG6[],load:0->NXM_NX_REG7[],load:0->NXM_NX_REG8[],load:0->NXM_NX_REG9[],load:0->NXM_OF_IN_PORT[],resubmit(,16))

 cookie=0x0, table=65, priority=100,reg15=0x1,metadata=0x1 actions=clone(ct_clear,load:0->NXM_NX_REG11[],load:0->NXM_NX_REG12[],load:0->NXM_NX_REG13[],load:0x7->NXM_NX_REG11[],load:0x5->NXM_NX_REG12[],load:0x2->OXM_OF_METADATA[],load:0x1->NXM_NX_REG14[],load:0->NXM_NX_REG10[],load:0->NXM_NX_REG15[],load:0->NXM_NX_REG0[],load:0->NXM_NX_REG1[],load:0->NXM_NX_REG2[],load:0->NXM_NX_REG3[],load:0->NXM_NX_REG4[],load:0->NXM_NX_REG5[],load:0->NXM_NX_REG6[],load:0->NXM_NX_REG7[],load:0->NXM_NX_REG8[],load:0->NXM_NX_REG9[],load:0->NXM_OF_IN_PORT[],resubmit(,16))

 cookie=0x0, table=65, priority=100,reg15=0x1,metadata=0x3 actions=clone(ct_clear,load:0->NXM_NX_REG11[],load:0->NXM_NX_REG12[],load:0->NXM_NX_REG13[],load:0x3->NXM_NX_REG11[],load:0x4->NXM_NX_REG12[],load:0x1->OXM_OF_METADATA[],load:0x2->NXM_NX_REG14[],load:0->NXM_NX_REG10[],load:0->NXM_NX_REG15[],load:0->NXM_NX_REG0[],load:0->NXM_NX_REG1[],load:0->NXM_NX_REG2[],load:0->NXM_NX_REG3[],load:0->NXM_NX_REG4[],load:0->NXM_NX_REG5[],load:0->NXM_NX_REG6[],load:0->NXM_NX_REG7[],load:0->NXM_NX_REG8[],load:0->NXM_NX_REG9[],load:0->NXM_OF_IN_PORT[],resubmit(,16))



 //到本地某个虚拟机的直接发送

 cookie=0x0, table=65, priority=100,reg15=0x2,metadata=0x2 actions=output:3

 cookie=0x0, table=65, priority=100,reg15=0x2,metadata=0x3 actions=output:4



 //通过MAC_Binding修改IP对应的MAC

 cookie=0x0, table=66, priority=100,reg0=0xc0a8025c,reg15=0x2,metadata=0x1 actions=mod_dl_dst:52:54:00:c1:68:73

 cookie=0x0, table=66, priority=100,reg0=0xc0a8025b,reg15=0x2,metadata=0x1 actions=mod_dl_dst:52:54:00:c1:68:71

 cookie=0x0, table=66, priority=100,reg0=0xc0a8015b,reg15=0x1,metadata=0x1 actions=mod_dl_dst:52:54:00:c1:68:70

 cookie=0x0, table=66, priority=100,reg0=0,reg1=0,reg2=0,reg3=0,reg15=0x2,metadata=0x1 actions=mod_dl_dst:00:00:00:00:00:00

 cookie=0x0, table=66, priority=100,reg0=0,reg1=0,reg2=0,reg3=0,reg15=0x1,metadata=0x1 actions=mod_dl_dst:00:00:00:00:00:00
```

### 流量追踪

[https://www.cnblogs.com/weiduoduo/p/11142747.html](https://www.cnblogs.com/weiduoduo/p/11142747.html)

ping

```
[root@HikvisionOS ~]# ovs-appctl dpif/dump-flows br-int
recirc_id(0),in_port(3),ct_state(-new-est-rel-inv-trk),eth(src=02:d4:1d:8c:d9:9d,dst=02:d4:1d:8c:d9:9f),eth_type(0x0806),arp(sip=20.0.0.10,tip=20.0.0.1,op=1/0xff,sha=02:d4:1d:8c:d9:9d,tha=00:00:00:00:00:00), packets:0, bytes:0, used:never, actions:userspace(pid=4294963168,slow_path(action))
recirc_id(0x11d),in_port(5),ct_state(-new+est-rel-inv+trk),eth(src=02:d4:1d:8c:d9:9b,dst=02:d4:1d:8c:d9:9e),eth_type(0x0800),ipv4(src=10.0.0.10,dst=20.0.0.10,ttl=64,frag=no), packets:11, bytes:1078, used:0.296s, actions:ct_clear,ct_clear,set(eth(src=02:d4:1d:8c:d9:9f,dst=02:d4:1d:8c:d9:9d)),set(ipv4(src=10.0.0.10,dst=20.0.0.10,ttl=63)),3
recirc_id(0x11a),in_port(3),ct_state(-new+est-rel-inv+trk),eth(src=02:d4:1d:8c:d9:9e),eth_type(0x0800),ipv4(src=20.0.0.8/255.255.255.248,frag=no), packets:12, bytes:1176, used:0.296s, actions:ct(zone=9,nat),recirc(0x11b)
recirc_id(0),in_port(5),eth(src=02:d4:1d:8c:d9:9b),eth_type(0x0800),ipv4(src=10.0.0.10,dst=20.0.0.100,frag=no), packets:12, bytes:1176, used:0.296s, actions:ct(zone=9),recirc(0x117)
recirc_id(0x11b),in_port(3),eth(dst=02:d4:1d:8c:d9:9b),eth_type(0x0800),ipv4(dst=10.0.0.10,frag=no), packets:12, bytes:1176, used:0.296s, actions:5
recirc_id(0x11c),in_port(5),eth(src=02:d4:1d:8c:d9:9b,dst=02:d4:1d:8c:d9:9e),eth_type(0x0800),ipv4(dst=20.0.0.8/255.255.255.248,frag=no), packets:11, bytes:1078, used:0.296s, actions:ct(zone=9),recirc(0x119)
recirc_id(0),in_port(5),ct_state(-new-est-rel-inv-trk),eth(src=02:d4:1d:8c:d9:9b,dst=02:d4:1d:8c:d9:9e),eth_type(0x0806),arp(sip=10.0.0.10,tip=10.0.0.1,op=1/0xff,sha=02:d4:1d:8c:d9:9b,tha=00:00:00:00:00:00), packets:0, bytes:0, used:never, actions:userspace(pid=4294963166,slow_path(action))
recirc_id(0x119),in_port(5),ct_state(-new+est-rel-inv+trk),eth_type(0x0800),ipv4(frag=no), packets:11, bytes:1078, used:0.296s, actions:ct(zone=9,nat),recirc(0x11d)
recirc_id(0x117),in_port(5),ct_state(-new+est-rel-inv+trk),eth_type(0x0800),ipv4(frag=no), packets:11, bytes:1078, used:0.296s, actions:ct(zone=9,nat),recirc(0x11c)
recirc_id(0),in_port(3),ct_state(-new-est-rel-inv-trk),eth(src=02:d4:1d:8c:d9:9d,dst=02:d4:1d:8c:d9:9f),eth_type(0x0800),ipv4(src=20.0.0.10,dst=10.0.0.10,ttl=64,frag=no), packets:12, bytes:1176, used:0.296s, actions:ct_clear,ct_clear,set(eth(src=02:d4:1d:8c:d9:9e,dst=02:d4:1d:8c:d9:9b)),set(ipv4(src=20.0.0.10,dst=10.0.0.10,ttl=63)),ct(zone=9),recirc(0x11a)


ovs-appctl ofproto/trace br-int in_port=4,dl_src=02:d4:1d:8c:d9:9b,dl_dst=02:d4:1d:8c:d9:9e,ipv4,nw_src=10.0.0.10,nw_dst=20.0.0.100,nw_proto=1,icmp_type=0,icmp_code=0 -generate
# ovs-appctl ofproto/trace br-int in_port=4,dl_src=02:d4:1d:8c:d9:9b,dl_dst=02:d4:1d:8c:d9:9e,ipv4,nw_src=10.0.0.10,nw_dst=20.0.0.100,nw_proto=1,icmp_type=0,icmp_code=0 -generate
Flow: icmp,in_port=4,vlan_tci=0x0000,dl_src=02:d4:1d:8c:d9:9b,dl_dst=02:d4:1d:8c:d9:9e,nw_src=10.0.0.10,nw_dst=20.0.0.100,nw_tos=0,nw_ecn=0,nw_ttl=0,icmp_type=0,icmp_code=0
bridge("br-int")
----------------
0. in_port=4, priority 100
set_field:0x9->reg13
set_field:0x7->reg11
set_field:0xd->reg12
set_field:0x1->metadata
set_field:0x2->reg14
resubmit(,8)
8. reg14=0x2,metadata=0x1,dl_src=02:d4:1d:8c:d9:9b, priority 50, cookie 0x6047969c
resubmit(,9)
9. ip,reg14=0x2,metadata=0x1,dl_src=02:d4:1d:8c:d9:9b,nw_src=10.0.0.10, priority 90, cookie 0xb948ce75
resubmit(,10)
10. metadata=0x1, priority 0, cookie 0x5b23fa1b
resubmit(,11)
11. metadata=0x1, priority 0, cookie 0x85c5c31e
resubmit(,12)
12. ip,metadata=0x1,nw_dst=20.0.0.100, priority 100, cookie 0xdfbc9cba
load:0x1->NXM_NX_XXREG0[96]
resubmit(,13)
13. ip,reg0=0x1/0x1,metadata=0x1, priority 100, cookie 0xa5b7b054
ct(table=14,zone=NXM_NX_REG13[0..15])
drop
-> A clone of the packet is forked to recirculate. The forked pipeline will be resumed at table 14.
Final flow: icmp,reg0=0x1,reg11=0x7,reg12=0xd,reg13=0x9,reg14=0x2,metadata=0x1,in_port=4,vlan_tci=0x0000,dl_src=02:d4:1d:8c:d9:9b,dl_dst=02:d4:1d:8c:d9:9e,nw_src=10.0.0.10,nw_dst=20.0.0.100,nw_tos=0,nw_ecn=0,nw_ttl=0,icmp_type=0,icmp_code=0
Megaflow: recirc_id=0,eth,ip,in_port=4,vlan_tci=0x0000/0x1000,dl_src=02:d4:1d:8c:d9:9b,nw_src=10.0.0.10,nw_dst=20.0.0.100,nw_frag=no
Datapath actions: ct(zone=9),recirc(0x1cb)
===============================================================================
recirc(0x1cb) - resume conntrack with default ct_state=trk|new (use --ct-next to customize)
===============================================================================
Flow: recirc_id=0x1cb,ct_state=new|trk,ct_zone=9,eth,icmp,reg0=0x1,reg11=0x7,reg12=0xd,reg13=0x9,reg14=0x2,metadata=0x1,in_port=4,vlan_tci=0x0000,dl_src=02:d4:1d:8c:d9:9b,dl_dst=02:d4:1d:8c:d9:9e,nw_src=10.0.0.10,nw_dst=20.0.0.100,nw_tos=0,nw_ecn=0,nw_ttl=0,icmp_type=0,icmp_code=0
bridge("br-int")
----------------
thaw
Resuming from table 14
14. metadata=0x1, priority 0, cookie 0x50063cdd
resubmit(,15)
15. metadata=0x1, priority 0, cookie 0xf31c70df
resubmit(,16)
16. metadata=0x1, priority 0, cookie 0x13c4db5f
resubmit(,17)
17. metadata=0x1, priority 0, cookie 0x78c30bb9
resubmit(,18)
18. ct_state=+new+trk,ip,metadata=0x1,nw_dst=20.0.0.100, priority 110, cookie 0x752ce65e
group:1
ct(commit,table=19,zone=NXM_NX_REG13[0..15],nat(dst=20.0.0.10))
nat(dst=20.0.0.10)
-> A clone of the packet is forked to recirculate. The forked pipeline will be resumed at table 19.
Final flow: unchanged
Megaflow: recirc_id=0x1cb,ct_state=+new-est-rel-inv+trk,eth,icmp,in_port=4,vlan_tci=0x0000/0x1fff,vlan_tci1=0x0000/0x1fff,dl_src=02:d4:1d:8c:d9:9b,dl_dst=02:d4:1d:8c:d9:9e,nw_src=10.0.0.10,nw_dst=20.0.0.100,nw_frag=no,icmp_type=0x0/0xff,icmp_code=0x0/0xff
Datapath actions: ct(commit,zone=9,nat(dst=20.0.0.10)),recirc(0x1cc)
===============================================================================
recirc(0x1cc) - resume conntrack with default ct_state=trk|new (use --ct-next to customize)
===============================================================================
Flow: recirc_id=0x1cc,ct_state=new|trk,ct_zone=9,eth,icmp,reg0=0x1,reg11=0x7,reg12=0xd,reg13=0x9,reg14=0x2,metadata=0x1,in_port=4,vlan_tci=0x0000,dl_src=02:d4:1d:8c:d9:9b,dl_dst=02:d4:1d:8c:d9:9e,nw_src=10.0.0.10,nw_dst=20.0.0.100,nw_tos=0,nw_ecn=0,nw_ttl=0,icmp_type=0,icmp_code=0
bridge("br-int")
----------------
thaw
Resuming from table 19
19. metadata=0x1, priority 0, cookie 0x5dc957e1
resubmit(,20)
20. metadata=0x1, priority 0, cookie 0x75f5bdfa
resubmit(,21)
21. metadata=0x1, priority 0, cookie 0xa21b1697
resubmit(,22)
22. metadata=0x1, priority 0, cookie 0x31cb2e34
resubmit(,23)
23. metadata=0x1, priority 0, cookie 0x3626ad6f
resubmit(,24)
24. metadata=0x1,dl_dst=02:d4:1d:8c:d9:9e, priority 50, cookie 0x502275b8
set_field:0x1->reg15
resubmit(,32)
32. priority 0
resubmit(,33)
33. reg15=0x1,metadata=0x1, priority 100
set_field:0x7->reg11
set_field:0xd->reg12
resubmit(,34)
34. priority 0
set_field:0->reg0
set_field:0->reg1
set_field:0->reg2
set_field:0->reg3
set_field:0->reg4
set_field:0->reg5
set_field:0->reg6
set_field:0->reg7
set_field:0->reg8
set_field:0->reg9
resubmit(,40)
40. ip,metadata=0x1, priority 100, cookie 0x14cc5da4
load:0x1->NXM_NX_XXREG0[96]
resubmit(,41)
41. metadata=0x1, priority 0, cookie 0x65381f07
resubmit(,42)
42. ip,reg0=0x1/0x1,metadata=0x1, priority 100, cookie 0x65dbb075
ct(table=43,zone=NXM_NX_REG13[0..15])
drop
-> A clone of the packet is forked to recirculate. The forked pipeline will be resumed at table 43.
Final flow: recirc_id=0x1cc,eth,icmp,reg0=0x1,reg11=0x7,reg12=0xd,reg13=0x9,reg14=0x2,reg15=0x1,metadata=0x1,in_port=4,vlan_tci=0x0000,dl_src=02:d4:1d:8c:d9:9b,dl_dst=02:d4:1d:8c:d9:9e,nw_src=10.0.0.10,nw_dst=20.0.0.100,nw_tos=0,nw_ecn=0,nw_ttl=0,icmp_type=0,icmp_code=0
Megaflow: recirc_id=0x1cc,eth,ip,in_port=4,dl_src=02:d4:1d:8c:d9:9b,dl_dst=02:d4:1d:8c:d9:9e,nw_dst=20.0.0.64/26,nw_frag=no
Datapath actions: ct(zone=9),recirc(0x1cd)
===============================================================================
recirc(0x1cd) - resume conntrack with default ct_state=trk|new (use --ct-next to customize)
===============================================================================
Flow: recirc_id=0x1cd,ct_state=new|trk,ct_zone=9,eth,icmp,reg0=0x1,reg11=0x7,reg12=0xd,reg13=0x9,reg14=0x2,reg15=0x1,metadata=0x1,in_port=4,vlan_tci=0x0000,dl_src=02:d4:1d:8c:d9:9b,dl_dst=02:d4:1d:8c:d9:9e,nw_src=10.0.0.10,nw_dst=20.0.0.100,nw_tos=0,nw_ecn=0,nw_ttl=0,icmp_type=0,icmp_code=0
bridge("br-int")
----------------
thaw
Resuming from table 43
43. metadata=0x1, priority 0, cookie 0x441f8496
resubmit(,44)
44. metadata=0x1, priority 0, cookie 0x10069659
resubmit(,45)
45. metadata=0x1, priority 0, cookie 0xe5a2272f
resubmit(,46)
46. metadata=0x1, priority 0, cookie 0xdfdd721e
resubmit(,47)
47. metadata=0x1, priority 0, cookie 0x103a342b
resubmit(,48)
48. metadata=0x1, priority 0, cookie 0x49deb0bb
resubmit(,49)
49. reg15=0x1,metadata=0x1, priority 50, cookie 0x74ad6dec
resubmit(,64)
64. priority 0
resubmit(,65)
65. reg15=0x1,metadata=0x1, priority 100
clone(ct_clear,set_field:0->reg11,set_field:0->reg12,set_field:0->reg13,set_field:0x5->reg11,set_field:0xb->reg12,set_field:0x3->metadata,set_field:0x2->reg14,set_field:0->reg10,set_field:0->reg15,set_field:0->reg0,set_field:0->reg1,set_field:0->reg2,set_field:0->reg3,set_field:0->reg4,set_field:0->reg5,set_field:0->reg6,set_field:0->reg7,set_field:0->reg8,set_field:0->reg9,set_field:0->in_port,resubmit(,8))
ct_clear
set_field:0->reg11
set_field:0->reg12
set_field:0->reg13
set_field:0x5->reg11
set_field:0xb->reg12
set_field:0x3->metadata
set_field:0x2->reg14
set_field:0->reg10
set_field:0->reg15
set_field:0->reg0
set_field:0->reg1
set_field:0->reg2
set_field:0->reg3
set_field:0->reg4
set_field:0->reg5
set_field:0->reg6
set_field:0->reg7
set_field:0->reg8
set_field:0->reg9
set_field:0->in_port
resubmit(,8)
8. reg14=0x2,metadata=0x3,dl_dst=02:d4:1d:8c:d9:9e, priority 50, cookie 0x4a6a2617
resubmit(,9)
9. ip,metadata=0x3,nw_ttl=0, priority 30, cookie 0xcb4904dc
drop
Final flow: unchanged
Megaflow: recirc_id=0x1cd,ct_state=+new-est-rel-inv+trk,eth,ip,in_port=4,vlan_tci=0x0000/0x1000,dl_src=00:00:00:00:00:00/01:00:00:00:00:00,dl_dst=02:d4:1d:8c:d9:9e,nw_src=10.0.0.10,nw_dst=20.0.0.64/26,nw_ttl=0,nw_frag=no
Datapath actions: ct_clear
```

OVN Trace

```
$ sudo ovn-trace --minimal sw0 'inport == "sw0-port1" && eth.src == 00:00:00:00:00:01 && eth.dst == 00:00:00:00:00:02'
# reg14=0x1,vlan_tci=0x0000,dl_src=00:00:00:00:00:01,dl_dst=00:00:00:00:00:02,dl_type=0x0000
output("sw0-port2");

$ ovn-trace --summary sw0 'inport == "sw0-port1" && eth.src == 00:00:00:00:00:01 && eth.dst == 00:00:00:00:00:02'
# reg14=0x1,vlan_tci=0x0000,dl_src=00:00:00:00:00:01,dl_dst=00:00:00:00:00:02,dl_type=0x0000
ingress(dp="sw0", inport="sw0-port1") {
    next;
    outport = "sw0-port2";
    output;
    egress(dp="sw0", inport="sw0-port1", outport="sw0-port2") {
        output;
        /* output to "sw0-port2", type "" */;
    };
};

$ ovn-trace --detailed sw0 'inport == "sw0-port1" && eth.src == 00:00:00:00:00:01 && eth.dst == 00:00:00:00:00:02'
# reg14=0x1,vlan_tci=0x0000,dl_src=00:00:00:00:00:01,dl_dst=00:00:00:00:00:02,dl_type=0x0000

ingress(dp="sw0", inport="sw0-port1")
-------------------------------------
 0. ls_in_port_sec_l2 (ovn-northd.c:2979): inport == "sw0-port1" && eth.src == {00:00:00:00:00:01}, priority 50, uuid 50dd1db0
    next;
13. ls_in_l2_lkup (ovn-northd.c:3274): eth.dst == 00:00:00:00:00:02, priority 50, uuid faab2844
    outport = "sw0-port2";
    output;

egress(dp="sw0", inport="sw0-port1", outport="sw0-port2")
---------------------------------------------------------
 8. ls_out_port_sec_l2 (ovn-northd.c:3399): outport == "sw0-port2" && eth.dst == {00:00:00:00:00:02}, priority 50, uuid 4b4d798e
    output;
    /* output to "sw0-port2", type "" */

$ ovn-trace --detailed sw0 'inport == "sw0-port1" && eth.src == 00:00:00:00:00:ff && eth.dst == 00:00:00:00:00:02'
# reg14=0x1,vlan_tci=0x0000,dl_src=00:00:00:00:00:ff,dl_dst=00:00:00:00:00:02,dl_type=0x0000

ingress(dp="sw0", inport="sw0-port1")
-------------------------------------
 0. ls_in_port_sec_l2: no match (implicit drop)
```



