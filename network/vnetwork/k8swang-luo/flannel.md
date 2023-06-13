## Flannel

Flannel 主要提供的是集群内的 Overlay 网络，支持三种后端实现，分别是：UDP 模式、VXLAN 模式、host-gw 模式。

### UDP 模式

UDP 模式，是 Flannel 项目最早支持的一种方式，却也是性能最差的一种方式。这种模式提供的是一个三层的 Overlay 网络，即：它首先对发出端的 IP 包进行 UDP 封装，然后在接收端进行解封装拿到原始的 IP 包，进而把这个 IP 包转发给目标容器。工作原理如下图所示。

![](/assets/network-virtualnet-container-k8s-flannel1.png)node1 上的 pod1 请求 node2 上的 pod2 时，流量的走向如下：

1. pod1 里的进程发起请求，发出 IP 包；
2. IP 包根据 pod1 里的 veth 设备对，进入到 cni0 网桥；
3. 由于 IP 包的目的 ip 不在 node1 上，根据 flannel 在节点上创建出来的路由规则，进入到 flannel0 中；
4. 此时 flanneld 进程会收到这个包，flanneld 判断该包应该在哪台 node 上，然后将其封装在一个 UDP 包中；
5. 最后通过 node1 上的网关，发送给 node2；

flannel0 是一个 TUN 设备（Tunnel 设备）。在 Linux 中，TUN 设备是一种工作在三层（Network Layer）的虚拟网络设备。TUN 设备的功能：在操作系统内核和用户应用程序之间传递 IP 包。

可以看到，这种模式性能差的原因在于，整个包的 UDP 封装过程是 flanneld 程序做的，也就是用户态，而这就带来了一次内核态向用户态的转换，以及一次用户态向内核态的转换。在上下文切换和用户态操作的代价其实是比较高的，而 UDP 模式因为封包拆包带来了额外的性能消耗。

### VXLAN 模式

VXLAN，即 Virtual Extensible LAN（虚拟可扩展局域网），是 Linux 内核本身就支持的一种网络虚似化技术。通过利用 Linux 内核的这种特性，也可以实现在内核态的封装和解封装的能力，从而构建出覆盖网络。其工作原理如下图所示：

![](/assets/network-virtualnet-container-k8s-flannel2.png)VXLAN 模式的 flannel 会在节点上创建一个叫 flannel.1 的 VTEP \(VXLAN Tunnel End Point，虚拟隧道端点\) 设备，跟 UDP 模式一样，该设备将二层数据帧封装在 UDP 包里，再转发出去，而与 UDP 模式不一样的是，整个封装的过程是在内核态完成的。

node1 上的 pod1 请求 node2 上的 pod2 时，流量的走向如下：

1. pod1 里的进程发起请求，发出 IP 包；
2. IP 包根据 pod1 里的 veth 设备对，进入到 cni0 网桥；
3. 由于 IP 包的目的 ip 不在 node1 上，根据 flannel 在节点上创建出来的路由规则，进入到 flannel.1 中；
4. flannel.1 将原始 IP 包加上一个目的 MAC 地址，封装成一个二层数据帧；然后内核将数据帧封装进一个 UDP 包里；
5. 最后通过 node1 上的网关，发送给 node2；

### host-gw 模式

最后一种 host-gw 模式是一种纯三层网络方案。其工作原理为将每个 Flannel 子网的“下一跳”设置成了该子网对应的宿主机的 IP 地址，这台主机会充当这条容器通信路径里的“网关”。这样 IP 包就能通过二层网络达到目的主机，而正是因为这一点，host-gw 模式要求集群宿主机之间的网络是二层连通的，如下图所示。

![](/assets/network-virtualnet-container-k8s-flannel3.png)

宿主机上的路由信息是 flanneld 设置的，因为 flannel 子网和主机的信息保存在 etcd 中，所以 flanneld 只需要 watch 这些数据的变化，实时更新路由表即可。在这种模式下，容器通信的过程就免除了额外的封包和解包带来的性能损耗。

node1 上的 pod1 请求 node2 上的 pod2 时，流量的走向如下：

1. pod1 里的进程发起请求，发出 IP 包，从网络层进入链路层封装成帧；
2. 根据主机上的路由规则，数据帧从 Node 1 通过宿主机的二层网络到达 Node 2 上；





