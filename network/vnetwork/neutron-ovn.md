# Neutron OVN模式网络流量分析

### 东西向二层流量![](/assets/network-vnetwork-neutron-neutronovn.png)vlan网络net1下的云主机vm1访问vm3（上图标号1.\*）

vm1的流量到br-int走逻辑交换机，打上vlan标签从br-int和br-prv的patch口发出\(每个网络OVN都会自动创建一个patch口\)

```
 cookie=0xefb298f1, duration=359369.164s, table=65, n_packets=79, n_bytes=7086, idle_age=12740, hard_age=65534, priority=100,reg15=0x1,metadata=0x1f actions=mod_vlan_vid:1157,output:681,strip_vlan
```

外层交换机转发后数据从patch口进入后剥离vlan，并设置metadata等数据后转发到vm3

![](/assets/network-vnetwork-neutron-neutronovn2.png)









