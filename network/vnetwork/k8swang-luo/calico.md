## Calico

Calico 没有使用 CNI 的网桥模式，而是将节点当成边界路由器，组成了一个全连通的网络，通过 BGP 协议交换路由。所以，Calico 的 CNI 插件还需要在宿主机上为每个容器的 Veth Pair 设备配置一条路由规则，用于接收传入的 IP 包。

Calico 的组件：

1. CNI 插件：Calico 与 Kubernetes 对接的部分
2. Felix：负责在宿主机上插入路由规则（即：写入 Linux 内核的 FIB 转发信息库），以及维护 Calico 所需的网络设备等工作。
3. BIRD \(BGP Route Reflector\)：是 BGP 的客户端，专门负责在集群里分发路由规则信息。

三个组件都是通过一个 DaemonSet 安装的。CNI 插件是通过 initContainer 安装的；而 Felix 和 BIRD 是同一个 pod 的两个 container。

### 工作原理

Calico 采用的 BGP，就是在大规模网络中实现节点路由信息共享的一种协议。全称是 Border Gateway Protocol，即：边界网关协议。它是一个 Linux 内核原生就支持的、专门用在大规模数据中心里维护不同的 “自治系统” 之间路由信息的、无中心的路由协议。

由于没有使用 CNI 的网桥，Calico 的 CNI 插件需要为每个容器设置一个 Veth Pair 设备，然后把其中的一端放置在宿主机上，还需要在宿主机上为每个容器的 Veth Pair 设备配置一条路由规则，用于接收传入的 IP 包。如下图所示：

![](/assets/network-virtualnet-container-k8s-bgp1.png)

可以使用 calicoctl 查看 node1 的节点连接情况：

```
~ calicoctl get no
NAME
node1
node2
node3
~ calicoctl node status
Calico process is running.

IPv4 BGP status
+---------------+-------------------+-------+------------+-------------+
| PEER ADDRESS  |     PEER TYPE     | STATE |   SINCE    |    INFO     |
+---------------+-------------------+-------+------------+-------------+
| 192.168.50.11 | node-to-node mesh | up    | 2020-09-28 | Established |
| 192.168.50.12 | node-to-node mesh | up    | 2020-09-28 | Established |
+---------------+-------------------+-------+------------+-------------+

IPv6 BGP status
No IPv6 peers found.
```

可以看到整个 calico 集群上有 3 个节点，node1 和另外两个节点处于连接状态，模式为 “Node-to-Node Mesh”。再看下 node1 上的路由信息如下：

```
~ ip route
default via 192.168.50.1 dev ens33 proto static metric 100
10.244.104.0/26 via 192.168.50.11 dev ens33 proto bird
10.244.135.0/26 via 192.168.50.12 dev ens33 proto bird
10.244.166.128 dev cali717821d73f3 scope link
blackhole 10.244.166.128/26 proto bird
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1
192.168.50.0/24 dev ens33 proto kernel scope link src 192.168.50.10 metric 100
```

其中，第 2 条的路由规则表明 10.244.104.0/26 网段的数据包通过 bird 协议由 ens33 设备发往网关 192.168.50.11。这也就定义了目的 ip 为 node2 上 pod 请求的走向。第 3 条路由规则与之类似。

### IPIP 模式

IPIP 模式为了解决两个 node 不在一个子网的问题。只要将名为 calico-node 的 daemonset 的环境变量 CALICO\_IPV4POOL\_IPIP 设置为 "Always" 即可。如下：

```
- name: CALICO_IPV4POOL_IPIP
  value: "Off"
```

IPIP 模式的 calico 使用了 tunl0 设备，这是一个 IP 隧道设备。IP 包进入 tunl0 后，内核会将原始 IP 包直接封装在宿主机的 IP 包中；封装后的 IP 包的目的地址为下一跳地址，即 node2 的 IP 地址。由于宿主机之间已经使用路由器配置了三层转发，所以这个 IP 包在离开 node 1 之后，就可以经过路由器，最终发送到 node 2 上。如下图所示。

![](/assets/network-virtualnet-container-k8s-calico2.png)





