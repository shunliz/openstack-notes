# XDP {#dnqxk5}

# 摘要

近些年业界流行通过**内核旁路**（kernel bypass）的方式实现 可编程的包处理过程（programmable packet processing）。实现方式是 将网络硬件完全交由某个专门的**用户空间应用**（userspace application） 接管，从而避免内核和用户态上下文切换的昂贵性能开销。

但是，操作系统被旁路（绕过）之后，它的应用隔离（application isolation） 和安全机制（security mechanisms）就都失效了；一起失效的还有各种经过已经 充分测试的配置、部署和管理工具。

为解决这个问题，我们提出一种新的可编程包处理方式：eXpress Data Path \(**XDP**\)。

* XDP 提供了一个仍然基于操作系统内核的安全执行环境，在设备驱动上下文 （device driver context）中执行，可用于定制各种包处理应用。

* XDP 是主线内核（mainline Linux kernel）的一部分，与现有的内核 网络栈（kernel’s networking stack）完全兼容，二者协同工作。

* XDP 应用（application）通过 C 等高层语言编写，然后编译成特定字节码；出于安 全考虑，内核会首先对这些字节码执行静态分析，然后再将它们翻译成 处理器原生指令（native instructions）。

* 测试结果显示，XDP 能达到 24Mpps/core 的处理性能。

XDP（eXpress Data Path）为Linux内核提供了高性能、可编程的网络数据路径。由于网络包在还未进入网络协议栈之前就处理，它给Linux网络带来了巨大的性能提升（性能比DPDK还要高）。



为展示 XDP 灵活的编程模型，本文还将给出三个程序示例，

1. **layer-3 routing**（三层路由转发）

2. **inline DDoS protection**（DDoS 防护）

3. **layer-4 load balancing**（四层负载均衡）

![](/assets/network-virtualnet-inuxnet-xdp21.png)

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



# 1 引言

软件实现高性能包处理的场景，对每个包的处理耗时有着极高的要求。通用目的操作系统中 的网络栈更多是针对灵活性的优化，这意味着它们花在每个包上 的指令太多了，不适合网络高吞吐的场景。

因此，随后出现了一些专门用于包处理的软件开发工具，例如 Data Plane Development Kit \(DPDK\) \[16\]。这些工具一般都会完全绕过内核，将网络硬件直接交 给用户态的网络应用，并需要独占一个或多个 CPU。

## 1.1 现有方案（kernel bypass）存在的问题

内核旁路方式可以显著提升性能，但缺点也很明显：

* 很难与现有系统集成；

* 上层应用必须要将内核中已经非常成熟的模块在用户态重新实现一遍，例如路由表、高层协议栈等；

* 最坏的情况下，这种包处理应用只能工作在一个完全隔绝的环境，因为内核提供的常见工具和部署方式在这种情况下都不可用了。

* 导致系统越来越复杂，而且破坏了操作系统内核在把控的安全边界。 在基础设施逐渐迁移到 Kubernetes/Docker 等容器环境的背景下，这一点显得尤其严重， 因为在这种场景下，内核担负着资源抽象和隔离的重任。

## 1.2 新方案：给内核网络栈添加可编程能力

对此，本文提供了另一种解决方案：给内核网络栈添加可编程能力。这使得我们能在 兼容各种现有系统、复用已有网络基础设施的前提下，仍然实现高速包处理。 这个框架称为 XDP，

* XDP 定义了一个受限的执行环境（a limited execution environment），运行在一个 eBPF 指令虚拟机中。eBPF 是 BSD Packet Filter \(BPF\) \[37\] 的扩展。

* XDP 程序运行在内核上下文中，此时内核自身都还没有接触到包数据（ before the kernel itself touches the packet data），这使得我们能在网卡收到包后 最早能处理包的位置，做一些自定义数据包处理（包括重定向）。

* 内核在加载（load）时执行静态校验，以确保用户提供的 XDP 程序的安全。

* 之后，程序会被动态编译成原生机器指令（native machine instructions），以获得高性能。

XDP 已经在过去的几个内核 release 中逐渐合并到内核，但在本文之前，还没有关于 XDP 系统的 完整架构介绍。本文将对 XDP 做一个高层介绍。

## 1.3 新方案（XDP）的优点

我们的测试结果显示 XDP 能取得 `24Mpps/core` 的处理性能，这虽然与 DPDK 还有差距， 但相比于后者这种 kernel bypass 的方式，XDP 有非常多的优势。

具体地，XDP：

1. 与内核网络栈协同工作，将硬件的控制权完全留在内核范围内。带来的好处：

* 保持了内核的安全边界

* 无需对网络配置或管理工具做任何修改

无需任何特殊硬件特性，任何有 Linux 驱动的网卡都可以支持， 现有的驱动只需做一些修改，就能支持 XDP hooks。

可以选择性地复用内核网络栈中的现有功能，例如路由表或 TCP/IP 协议栈，在保持配置接口不变的前提下，加速关键性能路径（critical performance paths）。

保证 eBPF 指令集和 XDP 相关的编程接口（API）的稳定性。

与常规 socket 层交互时，没有从用户态将包重新注入内核的昂贵开销。

对应用透明。这创造了一些新的部署场景/方式，例如直接在应用所 在的服务器上部署 DoS 防御（而非中心式/网关式 DoS 防御）。

服务不中断的前提下动态重新编程（dynamically re-program）， 这意味着可以按需加入或移除功能，而不会引起任何流量中断，也能动态响应系统其他部分的的变化。

无需预留专门的 CPU 做包处理，这意味着 CPU 功耗与流量高低直接相关，更节能。

## 1.4 本文组织结构

接下来的内容介绍 XDP 的设计，并做一些性能分析。结构组织如下：

* **Section 2 **介绍相关工作；

* **Section 3** 介绍 XDP 系统的设计；

* **Section 4** 做一些性能分析；

* **Section 5** 提供了几个真实 XDP 场景的程序例子；

* **Section 6** 讨论 XDP 的未来发展方向；

* **Section 7** 总结。

# 2 相关工作

**XDP**当然不是第一个支持可编程包处理的系统 —— 这一领域在过去几年发展势头良好， 并且趋势还在持续。业内已经有了几种可编程包处理框架，以及基于这些框架的新型应用，包括：

* 单一功能的应用，如 switching \[47\], routing \[19\], named-based forwarding \[28\], classification \[48\], caching \[33\] or traffic generation \[14\]。

* 更加通用、且高度可定制的包处理解决方案，能够处理从多种源收来的数据包 \[12, 20, 31, 34, 40, 44\]。

要基于通用（Common Off The Shelf，COTS）硬件实现高性能包处理，就必须解决 网卡（NIC）和包处理程序之间的所有瓶颈。由于性能瓶颈主要来源于内核和用户态应用之间的接口（系统调用开销非常大，另外，内核功能丰富，但也非常复杂）， 低层（low-level）框架必须通过这样或那样的方式来降低这些开销。

现有的一些框架通过几种不同的方式实现了高性能，XDP构建在其中一些技术之上。 接下来对 XDP 和它们的异同做一些比较分析。

## 2.1 用户态轮询 vs. XDP

* **DPDK** \[16\] 可能是使用最广泛的高性能包处理框架。它最初只支持 Intel 网卡，后来逐 步扩展到其他厂商的网卡。DPDK 也称作内核旁路框架（kernel bypass framework）， 因为它将网络硬件的控制权从内核转移到了用户态的网络应用，完全避免了内核-用户态 之间的切换开销。

* 与DPDK 类似的还有 **PF\_RING** ZC module \[45\] 和 hardware-specific Solarflare OpenOnload \[24\]。

在现有的所有框架中，内核旁路方式性能是最高的 \[18\]；但如引言中指 出，这种方式在管理、维护和安全方面都存在不足。

XDP 采用了一种与内核旁路截然相反的方式：相比于将网络硬件的控制权上移到用户空间， XDP 将性能攸关的包处理操作直接放在内核中，在操作系统的网络栈之前执行。

* 这种方式同样避免了内核-用户态切换开销（所有操作都在内核）；

* 但仍然由内核来管理硬件，因此保留了操作系统提供的管理接口和安全防护能力；

这里的主要创新是：使用了一个虚拟执行环境，它能对加载的 程序进行校验，确保它们不会对内核造成破坏。

## 2.2 内核模块 vs. XDP

在 XDP 之前，以内核模块（kernel module）方式实现包处理功能代价非常高， 因为程序执行出错时可能会导致整个系统崩溃，而且内核的内部 API 也会随着时间发生变化。 因此也就不难理解为什么只有很少的系统采用了这种方式。其中做的比较好包括

* 虚拟交换机 Open vSwitch \[44\]

* Click \[40\]

* 虚拟机路由器 Contrail \[41\]

这几个系统都支持灵活的配置，适用于多种场景，取得比较小的平摊代价。

XDP通过：

1. 提供一个安全的执行环境，以及内核社区支持，提供与那些暴露到用户空间一样稳定的内核 API

   极大地降低了那些将处理过程下沉到内核的应用（applications of moving processing into the kernel）的成本。

2. 此外，XDP 程序也能够完全绕过内核网络栈（completely bypass）， 与在内核网络栈中做 hook 的传统内核模块相比，性能也更高。

3. XDP 除了能将处理过程下沉到内核以获得最高性能之外，还支持在程序中执行重定向 （redirection）操作，完全绕过内核网络栈，将包送到特殊类型的用户空间 socket； 甚至能工作在 zero-copy 模式，进一步降低开销。

* 这种模式与 Netmap \[46\] 和 PF\_RING \[11\] 方式类似，但后者是在没有完全绕过内 核的情况下，通过降低从网络设备到用户态应用（network device to userspace application）之间的传输开销，实现高性能包处理。

内核模块方式的另一个例子是 Packet I/O engine，这是 PacketShader \[19\] 的组成部分， 后者专用于 Arrakis \[43\] and ClickOS \[36\] 之类的特殊目的操作系统。

## 2.3 可编程硬件 vs. XDP

可编程硬件设备也是一种实现高性能包处理的方式。

* 一个例子是 NetFPGA \[32\]，通过对它暴露的 API 进行编程，能够在这种基于 FPGA 的专 用设备上运行任何包处理任务。

* P4 编程语言 \[7\] 致力于将这种可编程能力扩展到更广泛的包处理硬件上 （巧合的是，它还包括了一个 XDP backend \[51\]）。

某种意义上来说，XDP 可以认为是一种 **offload** 方式：

1. 性能敏感的处理逻辑下放到网卡驱动中，以提升性能；

2. 其他的处理逻辑仍然走内核网络栈；

3. 如果没有用到内核 helper 函数，那整个 XDP 程序都可以 offload 到网卡（目前 Netronome smart-NICs \[27\] 已经支持）。

## 2.4 小结

XDP 提供了一种高性能包处理方式，与已有方式相比，在性能、与现有系统的集成、灵活性 等方面取得了更好的平衡。接下来介绍 XDP 是如何取得这种平衡的。

# 3 XDP 设计

XDP 的设计理念：

* **高性能包处理**

* **集成到操作系统内核（kernel）并与之协同工作**，同时

* 确保系统其它部分的**安全性（safety）**和**完整性（integrity）**

这种与内核的深度集成显然会给设计带来一些限制，在 XDP 组件合并到 Linux 的过程中，我们也收到了许多来自社区的反馈，促使我们不断调整 XDP 的设计，但 这些设计反思不在本文讨论范围之内。

## 3.0 XDP 系统架构

图 1 描绘了整个 XDP 系统，四个主要组成部分：

1. XDP driver hook：XDP 程序的主入口，在网卡收到包执行。

2. eBPF virtual machine：执行 XDP 程序的字节码，以及对字节码执行 JIT 以提升性能。

3. BPF maps：内核中的 key/value 存储，作为图中各系统的主要通信通道。

4. eBPF verifier：加载程序时对其执行静态验证，以确保它们不会导致内核崩溃。

![](https://img-blog.csdnimg.cn/img_convert/4ced6bfa0cef9d0b094a589649adaffe.png "4ced6bfa0cef9d0b094a589649adaffe.png")

Fig 1. XDP 与 Linux 网络栈的集成。这里只画了 ingress 路径，以免图过于复杂。

上图是 ingress 流程。网卡收到包之后，在处理包数据（packet data）之前，会先执行 main XDP hook 中的 eBPF 程序。 这段程序可以选择：

1. 丢弃（drop）这个包；或者

2. 通过当前网卡将包再发送（send）出去；或者

3. 将包重定向（redirect）到其他网络接口（包括虚拟机的虚拟网卡），或者通过 AF\_XDP socket 重定向到用户空间；或者

4. 放行（allow）这个包，如果后面没有其他原因导致的 drop，这个包就会进入常规的内核网络栈。如果是这种情况，也就是放行包进入内核网络栈，那接下来在将包放到发送队列之前（before packets are queued for transmission）， 还有一个能执行 BPF 程序的地方：TC BPF hook。

此外，图 1 中还可以看出，不同的 eBPF 程序之间、eBPF 程序和用户空间应用之间，都能够通过 BPF maps 进行通信。

## 3.1 XDP driver hook

### 在设备驱动中执行，无需上下文切换

XDP 程序在网络设备驱动中执行，网络设备每收到一个包，程序就执行一次。

相关代码实现为一个内核库函数（library function），因此程序直接 在设备驱动中执行，无需切换到用户空间上下文。

### 在软件最早能处理包的位置执行，性能最优

回到上面图 1 可以看到：程序在网卡收到包之后最早能处理包的位置 执行 —— 此时内核还没有为包分配 struct sk\_buff 结构体， 也没有执行任何解析包的操作。

### XDP 程序典型执行流

下图是一个典型的 XDP 程序执行流：



## 参考文档 {#fdsrev}

* [Introduction to XDP](https://www.iovisor.org/technology/xdp)
* [Network Performance BoF](http://people.netfilter.org/hawk/presentations/NetDev1.1_2016/links.html)
* [XDP Introduction and Use-cases](http://people.netfilter.org/hawk/presentations/xdp2016/xdp_intro_and_use_cases_sep2016.pdf)
* [Linux Network Stack](http://people.netfilter.org/hawk/presentations/theCamp2016/theCamp2016_next_steps_for_linux.pdf)
* [NetDev 1.2 video](https://www.youtube.com/watch?v=NlMQ0i09HMU&feature=youtu.be&t=3m3s)



