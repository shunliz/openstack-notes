# XDP {#dnqxk5}

XDP（eXpress Data Path）为Linux内核提供了高性能、可编程的网络数据路径。由于网络包在还未进入网络协议栈之前就处理，它给Linux网络带来了巨大的性能提升（性能比DPDK还要高）。![](/assets/network-virtualnet-inuxnet-xdp21.png)

![](/assets/network-virtualnet-linuxnet-xdp1.png)

XDP主要的特性包括

* 在网络协议栈前处理
* 无锁设计
* 批量I/O操作
* 轮询式
* 直接队列访问
* 不需要分配skbuff
* 支持网络卸载
* DDIO
* XDP程序快速执行并结束，没有循环
* Packeting steering

## 与DPDK对比 {#fav5d}

相对于DPDK，XDP具有以下优点

* 无需第三方代码库和许可
* 同时支持轮询式和中断式网络
* 无需分配大页
* 无需专用的CPU
* 无需定义新的安全网络模型

## 示例 {#g42nnb}

* [Linux内核BPF示例](https://github.com/torvalds/linux/tree/master/samples/bpf)
* [prototype-kernel示例](https://github.com/netoptimizer/prototype-kernel/tree/master/kernel/samples/bpf)
* [libbpf](https://github.com/torvalds/linux/tree/master/tools/lib/bpf)

## 缺点 {#c4d4sv}

注意XDP的性能提升是有代价的，它牺牲了通用型和公平性

* XDP不提供缓存队列（qdisc），TX设备太慢时直接丢包，因而不要在RX比TX快的设备上使用XDP
* XDP程序是专用的，不具备网络协议栈的通用性

# XDP架构 {#7p4i8f}

XDP基于一系列的技术来实现高性能和可编程性，包括

* 基于eBPF
* Capabilities negotiation：通过协商确定网卡驱动支持的特性，XDP尽量利用新特性，但网卡驱动不需要支持所有的特性
* 在网络协议栈前处理
* 无锁设计
* 批量I/O操作
* 轮询式
* 直接队列访问
* 不需要分配skbuff
* 支持网络卸载
* DDIO
* XDP程序快速执行并结束，没有循环
* Packeting steering

## 包处理逻辑 {#1s3896}

如下图所示，基于内核的eBPF程序处理包，每个RX队列分配一个CPU，且以每个网络包一个Page（packet-page）的方式避免分配skbuff。

![](/assets/network-virtualnet-linuxnet-xdp2.png)

# XDP使用场景 {#h1-xdp-}

XDP的使用场景包括

* DDoS防御
* 防火墙
* 基于
  `XDP_TX`
  的负载均衡
* 网络统计
* 复杂网络采样
* 高速交易平台

## 参考文档 {#fdsrev}

* [Introduction to XDP](https://www.iovisor.org/technology/xdp)
* [Network Performance BoF](http://people.netfilter.org/hawk/presentations/NetDev1.1_2016/links.html)
* [XDP Introduction and Use-cases](http://people.netfilter.org/hawk/presentations/xdp2016/xdp_intro_and_use_cases_sep2016.pdf)
* [Linux Network Stack](http://people.netfilter.org/hawk/presentations/theCamp2016/theCamp2016_next_steps_for_linux.pdf)
* [NetDev 1.2 video](https://www.youtube.com/watch?v=NlMQ0i09HMU&feature=youtu.be&t=3m3s)



