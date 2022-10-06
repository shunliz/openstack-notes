# iptables/netfilter {#ei4679}

iptables是一个配置Linux内核防火墙的命令行工具，它基于内核的netfilter机制。新版本的内核（3.13+）也提供了nftables，用于取代iptables。

## netfilter {#7uff4o}

netfilter是Linux内核的包过滤框架，它提供了一系列的钩子（Hook）供其他模块控制包的流动。这些钩子包括

* `NF_IP_PRE_ROUTING`
  ：刚刚通过数据链路层解包进入网络层的数据包通过此钩子，它在路由之前处理
* `NF_IP_LOCAL_IN`
  ：经过路由查找后，送往本机（目的地址在本地）的包会通过此钩子
* `NF_IP_FORWARD`
  ：不是本地产生的并且目的地不是本地的包（即转发的包）会通过此钩子
* `NF_IP_LOCAL_OUT`
  ：所有本地生成的发往其他机器的包会通过该钩子
* `NF_IP_POST_ROUTING`
  ：在包就要离开本机之前会通过该钩子，它在路由之后处理

![](/assets/network-virtualnet-linuxnet-nfiptalbes1.png)

## iptables {#4n041j}

iptables通过表和链来组织数据包的过滤规则，每条规则都包括匹配和动作两部分。默认情况下，每张表包括一些默认链，用户也可以添加自定义的链，这些链都是顺序排列的。这些表和链包括：

* raw表用于决定数据包是否被状态跟踪机制处理，内建PREROUTING和OUTPUT两个链
* filter表用于过滤，内建INPUT（目的地是本地的包）、FORWARD（不是本地产生的并且目的地不是本地）和OUTPUT（本地生成的包）等三个链
* nat用表于网络地址转换，内建PREROUTING（在包刚刚到达防火墙时改变它的目的地址）、INPUT、OUTPUT和POSTROUTING（要离开防火墙之前改变其源地址）等链
* mangle用于对报文进行修改，内建PREROUTING、INPUT、FORWARD、OUTPUT和POSTROUTING等链
* security表用于根据安全策略处理数据包，内建INPUT、FORWARD和OUTPUT链

| Tables↓/Chains→ | PREROUTING | INPUT | FORWARD | OUTPUT | POSTROUTING |
| :--- | :--- | :--- | :--- | :--- | :--- |
| \(routing decision\) |  |  |  | ✓ |  |
| **raw** | ✓ |  |  | ✓ |  |
| \(connection tracking enabled\) | ✓ |  |  | ✓ |  |
| **mangle** | ✓ | ✓ | ✓ | ✓ | ✓ |
| **nat**\(DNAT\) | ✓ |  |  | ✓ |  |
| \(routing decision\) | ✓ |  |  | ✓ |  |
| **filter** |  | ✓ | ✓ | ✓ |  |
| **security** |  | ✓ | ✓ | ✓ |  |
| **nat**\(SNAT\) |  | ✓ |  |  | ✓ |

所有链默认都是没有任何规则的，用户可以按需要添加规则。每条规则都包括匹配和动作两部分：

* 匹配可以有多条，比如匹配端口、IP、数据包类型等。匹配还可以包括模块（如conntrack、recent等），实现更复杂的过滤。
* 动作只能有一个，通过
  `-j`
  指定，如ACCEPT、DROP、RETURN、SNAT、DNAT等

这样，网络数据包通过iptables的过程为

![](/assets/network-virtualnet-linuxnet-nfiptables2.png)

其规律为

1. 当一个数据包进入网卡时，数据包首先进入PREROUTING链，在PREROUTING链中我们有机会修改数据包的DestIP\(目的IP\)，然后内核的”路由模块”根据”数据包目的IP”以及”内核中的路由表” 判断是否需要转送出去\(注意，这个时候数据包的DestIP有可能已经被我们修改过了\)
2. 如果数据包就是进入本机的\(即数据包的目的IP是本机的网口IP\)，数据包就会沿着图向下移动，到达INPUT链。数据包到达INPUT链后，任何进程都会收到它
3. 本机上运行的程序也可以发送数据包，这些数据包经过OUTPUT链，然后到达POSTROTING链输出\(注意，这个时候数据包的SrcIP有可能已经被我们修改过了\)
4. 如果数据包是要转发出去的\(即目的IP地址不再当前子网中\)，且内核允许转发，数据包就会向右移动，经过FORWARD链，然后到达POSTROUTING链输出\(选择对应子网的网口发送出去\)

### iptables示例 {#1zhljm}

查看规则列表

```

```

允许22端口

```

```

允许来自192.168.0.4的包

```

```

允许现有连接或与现有连接关联的包

```

```

禁止ping包

```

```

禁止所有其他包

```

```

MASQUERADE

```

```

NAT

```

```

端口映射

```

```

重置所有规则

```

```

## nftables {#fank1g}



