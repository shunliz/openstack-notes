# Neutron OVN模式网络流量分析

原文地址：https://www.jianshu.com/p/44153cf101dd

### 东西向二层流量![](/assets/network-vnetwork-neutron-neutronovn.png)vlan网络net1下的云主机vm1访问vm3（上图标号1.\*）

vm1的流量到br-int走逻辑交换机，打上vlan标签从br-int和br-prv的patch口发出\(每个网络OVN都会自动创建一个patch口\)

```
 cookie=0xefb298f1, duration=359369.164s, table=65, n_packets=79, n_bytes=7086, idle_age=12740, hard_age=65534, priority=100,reg15=0x1,metadata=0x1f actions=mod_vlan_vid:1157,output:681,strip_vlan
```

外层交换机转发后数据从patch口进入后剥离vlan，并设置metadata等数据后转发到vm3

```
 cookie=0xefb298f1, duration=345143.445s, table=0, n_packets=74, n_bytes=7084, idle_age=65534, hard_age=65534, priority=150,in_port=1125,dl_vlan=1157 actions=strip_vlan,load:0x61->NXM_NX_REG13[],load:0x5f->NXM_NX_REG11[],load:0x60->NXM_NX_REG12[],load:0x1f->OXM_OF_METADATA[],load:0x1->NXM_NX_REG14[],resubmit(,8)
```

## ![](/assets/network-vnetwork-neutron-neutronovn2.png)东西向三层流量

![](/assets/network-vnetwork-neutron-neutronovn-l3ew.png)![](/assets/network-vnentwork-neutron-neutronovn-l3ew2.png)![](/assets/network-vnetwork-neutron-neutronovn-l3gg.png)

## 南北向流量SNAT模式

![](/assets/network-vnetwork-neutron-neutronovn-sn1.png)![](/assets/network-vnentwork-neutron-neutronovn-snnat.png)**geneve网络下虚机访问外部网络（lrp port和虚机在不同一节点）**

**vlan网络net1下的云主机vm1访问外部ip，但是逻辑路由的lrp port不在vm1所在的节点**

逻辑路由信息

```
()[root@ovn-ovsdb-nb-2 /]# ovn-nbctl show 9eb2766d-dec8-4f73-8bc8-9f31dab3d6db
router 9eb2766d-dec8-4f73-8bc8-9f31dab3d6db (neutron-62e2ca59-b6cf-4a17-a531-776fecd8dbf5) (aka lc_router)
    port lrp-f60cd81a-7a5e-4b2d-b687-fcf2881a725a
        mac: "fa:16:3e:fd:11:6c"
        networks: ["10.66.0.1/16"]
    port lrp-cc3da0db-31e8-40a0-ad9f-c54f7a848ac7
        mac: "fa:16:3e:e7:f2:d2"
        networks: ["172.90.0.111/24"]
        gateway chassis: [9e7fc81f-12dc-4e83-b28c-a90060c579c6 ef5bb610-b0e6-4dc4-aa59-6d4d7972bc22 e8427df8-1dc6-45e3-b45c-35c35e8d6ed3]
    port lrp-d14d66bc-dfda-44a2-b1aa-a0481c0119ab
        mac: "fa:16:3e:6c:a0:2e"
        networks: ["192.168.112.1/24"]
    nat 1b9b075b-e56d-45e7-a750-cb1e366a7d08
        external ip: "172.90.0.111"
        logical ip: "192.168.112.0/24"
        type: "snat"
    nat 1d0eb0c0-67e8-43a3-8f7d-4da54d3fa562
        external ip: "172.90.0.111"
        logical ip: "10.66.0.0/16"
        type: "snat"
    nat bc8472b4-3ef9-4a18-bc6b-e9f12db01d4d
        external ip: "172.90.0.118"
        logical ip: "10.66.3.139"
        type: "dnat_and_snat"
```

lrp-cc3da0db-31e8-40a0-ad9f-c54f7a848ac7落在的节点信息\(lrp port在node-3上，但是vm1在node-1\)

```
()[root@ovn-ovsdb-sb-0 /]# ovn-sbctl show | grep -B 9 lrp-cc3da0db-31e8-40a0-ad9f-c54f7a848ac7
Chassis "9e7fc81f-12dc-4e83-b28c-a90060c579c6"
    hostname: node-3.domain.tld
    Encap geneve
        ip: "192.168.20.4"
        options: {csum="true"}
    Port_Binding "a7578b9a-c0b6-4980-8961-c968e027926f"
    Port_Binding "a9dce74f-f655-42b8-9529-f26a370feca9"
    Port_Binding "3427338f-1e5a-494b-9a8c-34bc80590c61"
    Port_Binding "f57f5ac7-6491-416f-8eb5-e4d1cb5fd41f"
    Port_Binding cr-lrp-cc3da0db-31e8-40a0-ad9f-c54f7a848ac7
```

数据包路径\(vlan网络为例\):

```
vm1->br-int(node-1)->br-ex(node-1)->br-ex(node-3)->br-int(node-3)->br-ex(node-3)->public network->target ip->br-ex(node-3)->patch port to br-prv->private network->br-prv->patch port to br-int->br-int->vm1
```

1, vm1出来的数据包，先经过本节点的逻辑路由发往lrp port（sip: ip vm1, dip: dst ip, smac: mac chassis which vm1 on, dmac: mac lrp port）并带上public network的vlan号

```
16:48:17.822115 6a:22:16:11:a3:47 > fa:16:3e:e7:f2:d2, ethertype 802.1Q (0x8100), length 102: vlan 2901, p 0, ethertype IPv4, (tos 0x0, ttl 63, id 39850, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.112.13 > 14.215.177.39: ICMP echo request, id 56321, seq 25, length 64
```

2, lrp port所在节点的逻辑路由将sip和smac转换成lrp port的ip和mac后通过br-ex发出（sip: ip lrp port, dip: dst ip, smac: mac lrp port）

```
16:48:17.822127 fa:16:3e:e7:f2:d2 > f8:bc:12:4e:44:dd, ethertype 802.1Q (0x8100), length 102: vlan 2901, p 0, ethertype IPv4, (tos 0x0, ttl 62, id 39850, offset 0, flags [DF], proto ICMP (1), length 84)
    172.90.0.111 > 14.215.177.39: ICMP echo request, id 56321, seq 25, length 64
```

3, 数据回包时lrp port所在的逻辑路由收到数据包（sip: target ip, dip: ip lrp port, dmac: mac lrp port）后直接路由，将数据包打上vlan后通过br-prv发出（sip: target ip, dip: ip vm1, dmac: mac vm1）

```
16:48:17.858433 00:1d:09:65:eb:63 > fa:16:3e:e7:f2:d2, ethertype 802.1Q (0x8100), length 102: vlan 2901, p 0, ethertype IPv4, (tos 0x0, ttl 51, id 39850, offset 0, flags [DF], proto ICMP (1), length 84)
    14.215.177.39 > 172.90.0.111: ICMP echo reply, id 56321, seq 25, length 64
16:48:17.858890 f2:d0:90:45:36:48 > fa:16:3e:d7:41:89, ethertype 802.1Q (0x8100), length 102: vlan 1157, p 0, ethertype IPv4, (tos 0x0, ttl 50, id 39850, offset 0, flags [DF], proto ICMP (1), length 84)
    14.215.177.39 > 192.168.112.13: ICMP echo reply, id 56321, seq 25, length 64
```

![](/assets/network-vnentwork-neutron-neutronovn-sn2.png)**geneve网络下虚机访问外部网络（lrp port和虚机在同一节点）**

## 南北向流量floagintip（SNAT and DNAT）模式

![](/assets/network-vnetwork-neutron-neutronovn-snatdnat.png)

![](/assets/network-vnetwork-neutron-neutronovn-snatdnat2.png)**通过floatingip访问外部网络**

