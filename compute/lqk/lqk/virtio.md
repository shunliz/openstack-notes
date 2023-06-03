# ![](/assets/compute-lqkv-virtio1.png)Virtio协议架构![](/assets/compute-lqkv-virtio2.png)Virtio技术演进![](/assets/compute-lqkv-virtio4.png)**1.virtio-net驱动与设备:**最原始的virtio网络

**Virtio网络设备是一种虚拟的以太网卡**，支持多队列的网络包收发。熟悉virtio的读者应该知道，在virtio的架构中有

前后端之分。在virtio 网络中，所谓的前端即是虚拟机中的virtio-net网卡驱动。而后端的实现多种多样，后端的变化往往标志着virtio网络的演化。

图一中的后端即是QEMU的实现版本，也是最原始的virtio-net后端（设备）。virtio标准将其对于队列的抽象称为**Virtqueue**。**Vring**即是对Virtqueue的具体实现。**一个Virtqueue由一个Available Ring和Used Ring组成**。前者用于前端向后端发送数据，而后者反之。而在virtio网络中的TX/RX Queue均由一个**Virtqueue**实现。

所有的I/O通信架构都有**数据平面与控制平面**之分。而对于virtio来说，通过PCI传输协议实现的virtio控制平面正是为了确保**Vring**能够用于前后端正常通信，并且配置好自定义的设备特性。而**数据平面**正是使用这**些通过共享内存实现的Vring来实现虚拟机与主机之间的通信。**

_**举**例来说，当virtio-net驱动**发送网络数据包**时，会将数据放置于Available Ring中之后，会触发一次通知（Notification）。这时QEMU会接管控制，将此网络包传递到**TAP设备**。接着QEMU将数据放于Used Ring中，并发出一次通知，这次通知会触发虚拟中断的注入。虚拟机收到这个中断后，就会到Used Ring中取得后端已经放置的数据。至此一次发送操作就完成了。**接收网络数据包**的行为也是类似，只不过这次virtio-net驱动是将空的buffer放置于队列之中，以便后端将收到的数据填充完成而已。_



![](/assets/compute-lqk-virtio-virtio4.png)

![](/assets/compute-lqk-virtio-virtio5.png)

# **2.vhost-net：**处于内核态的后端 {#2.vhost-net%EF%BC%9A%E5%A4%84%E4%BA%8E%E5%86%85%E6%A0%B8%E6%80%81%E7%9A%84%E5%90%8E%E7%AB%AF}

QEMU实现的virtio网络后端带来的网络性能并不如意，究其原因是因为**频繁的上下文切换，低效的数据拷贝、线程间同步**等。于是，内核实现了一个新的virtio网络后端驱动，名为**vhost-net**。与之而来的是一套新的**vhost协议**。

**vhost协议**可以将允许VMM将virtio的数据面offload到另一个组件上，而这个组件正是vhost-net。在这套实现中，**QEMU和vhost-net内核驱动使用ioctl来交换vhost消息**，并且**用eventfd来实现前后端的通知**。当vhost-net内核驱动加载后，它会暴露一个字符设备在**/dev/vhost-net**。

```
/dev/vhost-net
```

而QEMU会打开并初始化这个字符设备，并**调用ioctl来与vhost-net进行控制面通信**，其内容包含virtio的特性协商，将虚拟机内存映射传递给vhost-net等。

**对比最原始的virtio网络实现**，控制平面在原有的基础上转变为vhost协议定义的**ioctl操作**（对于前端而言仍是通过PCI传输层协议暴露的接口），基于共享内存实现的Vring转变为virtio-net与vhost-net共享，数据平面的另一方转变为vhost-net，并且前后端通知方式也转为基于eventfd的实现。

如图2所示，可以注意到，vhost-net仍然通过读写TAP设备来与外界进行数据包交换。而读到这里的读者不禁要问，那虚拟机是如何与本机上的其他虚拟机与外界的主机通信的呢？答案就是通过类似Open vSwitch \(OVS\)之类的软件交换机实现的。OVS相关的介绍这里就不再赘述。

* 《[使用DPDK打开Open vSwitch（OvS） \*概述](https://rtoax.blog.csdn.net/article/details/108747601)》
* 《[Open vSwitch（OVS）文档](https://rtoax.blog.csdn.net/article/details/109005008)》
* 《[《深入浅出DPDK》读书笔记（十五）：DPDK应用篇（Open vSwitch（OVS）中的DPDK性能加速）](https://rtoax.blog.csdn.net/article/details/109371440)》

![](/assets/compute-lqk-virtio-virtio21.png)**3.vhost-user: **使用DPDK加速的后端

GitHub仓库：[https://github.com/openebs/vhost-user](https://github.com/openebs/vhost-user)

---

DPDK社区一直致力于加速数据中心的网络数据平面，而virtio网络作为当今云环境下数据平面必不可少的一环，自然是DPDK优化的方向。而vhost-user就是结合DPDK的各方面优化技术得到的用户态virtio网络后端。这些优化技术包括：处理器亲和性，巨页的使用，轮询模式驱动等。除了vhost-user，DPDK还有自己的**virtio PMD**作为高性能的前端，本文将以vhost-user作为重点介绍。

**基于vhost协议，DPDK设计了一套新的用户态协议，名为vhost-user协议**，这套协议允许qemu将virtio设备的网络包处理offload到任何DPDK应用中（例如OVS-DPDK）。

* **vhost-user协议和vhost协议最大的区别其实就是通信信道的区别。**
* **Vhost协议通过对vhost-net字符设备进行ioctl实现；**
* **vhost-user协议则通过unix socket进行实现。**

通过这个unix socket，vhost-user协议允许QEMU通过以下重要的操作来配置数据平面的offload：

* 1. 特性协商：virtio的特性与vhost-user新定义的特性都可以通过类似的方式协商，而所谓协商的具体实现就是QEMU接收vhost-user的特性，与自己支持的特性取交集。
* 2. 内存区域配置：QEMU配置好内存映射区域，vhost-user使用mmap接口来映射它们。
* 3. Vring配置：QEMU将Virtqueue的个数与地址发送给vhost-user，以便vhost-user访问。
* 4. 通知配置：
  **vhost-user仍然使用eventfd来实现前后端通知。**

基于DPDK的**Open vSwitch\(OVS-DPDK\)**一直以来就对vhost-user提供了支持，读者可以通过在OVS-DPDK上**创建vhost-user端口**来使用这种高效的用户态后端。



![](/assets/compute-lqk-virtio-virtio32.png)



