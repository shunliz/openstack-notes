DPU（ Data Processing Unit，数据处理器）是针对数据处理和以数据为中心的计算模型设计的硬件加速器。与 CPU 和 GPU 等其他硬件加速器不同，DPU 使用MIMD架构，支持大规模并行处理。

许多企业将 DPU 用于人工智能和大数据等超级计算业务。通过阅读本文可以了解 DPU 的作用与优缺点，以帮助企业选择是否需要在其数据中心中使用 DPU。

DPU 本质上是为处理数据中心传输的数据而设计的。它专注于数据传输、数据缩减、数据安全，支持数据分析、加密和压缩。DPU 支持更高效的数据存储，从而能够释放 CPU 资源，让 CPU 专注于应用程序处理。

DPU 承担了 CPU 的网络和通信工作负载。它将处理器内核、硬件加速器和高性能网络接口相结合，能够处理大规模的工作负载。这种架构的实现使 DPU 能够确保正确的数据以正确的格式快速到达正确的位置。

当把 DPU 置于以数据为中心的基础架构的核心时，DPU 还可以解决服务器节点效率低下的问题。它可以缓解数据蔓延，提供高可用性和可靠性，并确保大规模数据的快速可访问性和可共享性。DPU 可以用于云计算数据中心或驱动复杂AI、ML\[3\]‍和深度学习算法的超级计算机。

简单来说，DPU 是一种具有硬件加速功能的可编程设备，它包含三个关键要素：

1. 一种行业标准、高性能、软件可编程的多核 CPU，通常基于广泛使用的 Arm 架构，与其他 SoC（System-on-a-chip，片上系统）组件紧密耦合。
2. 一种高性能网络接口，能够以线速或网络其余部分的速度解析、处理和有效地将数据传输到 GPU 和 CPU。
3. 一套丰富的灵活且可编程的加速引擎，可提高人工智能和机器学习、安全和存储等应用程序的性能。

STH（ServeTheHome） 尝试将 SmartNIC、DPU 和 Exotic NIC（基于 FPGA 的解决方案）进行了分类：  
![](/assets/微信图片_20220925012417.png)

可以划归到 DPU 分类的产品有：

* **NVIDIA BlueField-2：**
  是一个高度集成的 DPU，集成 ConnectX-6 DX 网络适配器与 ARM 处理器核阵列。BlueField-2 DPU通过 ASAP2的网络加速方案以及完整的数据面及控制面卸载，可以高效、高性能的支持虚拟化、裸金属、边缘计算场景的快速部署，通过 SNAP机制为存储提供完整的端到端解决方案。集成了各种安全加速，可以为数据中心提供隔离、安全性和加解密加速功能，它集成的 ARM Core 可以运行基础设施层的虚拟化、管理监控等功能。
* **NITRO 系统：**
  Nitro 系统用于为 AWS EC2 实例类型提供网络硬件卸载、EBS 存储硬件卸载、NVMe 本地存储、远程直接内存访问（RDMA）、裸金属实例的硬件保护/固件验证以及控制 EC2 实例所需的所有业务逻辑等。
* **Fungible DPU：**
  Fungible DPU 采用通用多线程处理器，结合标准以太网和 PCIe 接口。其他硬件组件包括高性能的片上 Fabric、定制的内存系统、一套完整的灵活数据加速器、可编程网络硬件流水线、可编程 PCIe 硬件流水线。
* **Pensando DPU：**
  包括网络功能（交换和路由、L3 ECMP、L4 负载均衡、Overlay 网络、VXLAN、IP-NAT等）、安全功能（微分段、DoS 保护、IPsec 终止、TLS/DTLS 终止等）以及存储功能（NVMe over TCP/RoCEv2、压缩/解压、加密/解密、SHA3 重复数据删除、CRC64/32 校验和等）。
* **Intel IPU：**
  IPU 使用专用协议加速器加速基础设施功能，包括存储虚拟化、网络虚拟化和安全性。允许灵活的工作负载放置来提高数据中心利用率。
* **Marvell DPU：**Marvell OCTEON 10 集成 ARM Neoverse N2 内核、1Tb 的交换机，支持内联加密，基于 VPP 的硬件加速器可将数据包处理速度提高多达 5 倍，基于机器学习的硬件加速引擎比软件处理性能提升 100 倍。

# **DPU 的应用场景**

DPU 承担了 CPU 的网络和通信工作负载，从而能够释放出 CPU 资源，让 CPU 专注于应用程序处理。DPU专注于以数据为中心的工作负载场景，例如数据传输、数据缩减、数据安全和分析。DPU 芯片采用特殊设计，将处理器内核与硬件加速器相结合。这种设计使 DPU 比 GPU 芯片更通用。DPU 拥有自己的专用操作系统，这意味着可以将其资源与主操作系统的资源结合起来，执行加密、纠删码、压缩或解压缩等功能。

云和超大规模提供商一直是这项技术的最早采用者。以下是 DPU 主要的应用场景：

## **网络数据路径加速引擎**

正常情况下，一个嵌入式 CPU对数据包的处理速度是无法超越独立 CPU 的。但如果网卡足够强大和灵活，能够处理所有网络中的数据，嵌入式 CPU 用来做控制路径的初始化和异常情况处理，这样就可以加速数据处理速度。

网络数据路径加速引擎至少需要具备以下功能：

* 像 OVS（开放式虚拟交换机）一样对数据包进行解析、匹配和处理。
* 基于 Zero Touch RoCE 的 RDMA 数据传输加速。
* 通过 GPU-Direct 加速器绕过 CPU，将来自存储和其他 GPU 的数据通过网络直接传给 GPU。
* TCP 通信加速，包括 RSS、LRO、checksum 等操作。
* 网络虚拟化的 VXLAN、 Geneve Overlay 和 VTEP offload。
* 面向多媒体流、CDN（内容分发网络）和新的 4K / 8K IP 视频（如基于 ST 2110 规范的 RiverMax）的 “Packet Pacing” 流量整形加速。
* 电信 Cloud RAN 的精准时钟加速器，例如 5T for 5G（精准时钟调度 5G 无线报文传输技术）功能。
* 在线 IPSEC 和 TLS 加密加速，但不影响其它正在运行的加速操作。
* 支持 SR-IOV、VirtIO 和 PV（Para-Virtualization）等虚拟化。

### **虚拟化网络功能（Virtual Network Function\)**

#### （1）内核虚拟化网络（vhost-net）

在虚拟化网络的初期，以打通虚拟机（VM）间和与外部通信能力为主，对功能诉求远高于性能，虚拟交换机OVS（Open vSwitch）的最初版本也是基于操作系统Linux内核转发来实现的。

#### （2）用户空间DPDK虚拟化网络（vhost-user）

随着虚拟化网络的发展，虚拟机（VM）业务对网络带宽的要求越来越高，另外，英特尔和Linux基金会推出了DPDK（Data Plane Development Kit）开源项 目，实现了用户空间直接从网卡收发数据报文并进行多核快速处理的开发库，虚拟交换机OVS将数据转发平面通过DPDK支持了用户空间的数据转发，进而实现了转发带宽量级的提升。

![](/assets/compute-hardware-dpu-net1.png)

#### （3）高性能SR-IOV网络（SR-IOV）

在一些对网络有高性能需求的场景，如NFV业务部署，OVS-DPDK的数据转 发 方 式 ， 无 法 满 足 高性能网络的 需 求 ， 这 样 就 引 入 的 SR-IOV 透 传（passthrough）到虚拟机（VM）的部署场景。

![](/assets/compute-hardware-dpu-net2.png)

#### （4）Virtio硬件加速虚拟化网络（vDPA）

为了解决高性能SRIOV网络的热迁移问题，出现了很多做法和尝试，尚未形成统一的标准。在Redhat提出硬件vDPA架构之前，Mellanox实现了软件vDPA（即VF Relay）。

![](/assets/compute-hardware-dpu-net3.png)

### **隔离网络虚拟化**

在传统的网卡上做云平台虚拟化，Hypervisor以及对应的虚拟化网络的实现，都是在主机操作系统上实现的。这样如果黑客如果攻陷了Hypervisor并拿到 主机操作系统的root权限，就可以通过篡改虚拟化网络配置，来对租户网络进行攻击，甚至可以渗透到其它计算节点，进行更大范围的攻击。

![](https://mmbiz.qpic.cn/mmbiz_png/6wxrMAnfIoTbXOrYLEHr8M3PYNmYcwSTpOVBCsic3zKw5K037ibBaOlsI3aO8YORVSDpia8AMG9XMG16iatBJxYjwg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1 "图片")

引入DPU智能网卡之后，将虚拟化网络的控制平面完全卸载到智能网卡 上，与主机操作系统相隔离。即使黑客攻陷了Hypervisor，获取了主机操作系统的root权限，也无法篡改虚拟化网络的配置，这样可以将黑客的攻击范围限制在 主机操作系统上，不会影响到虚拟化网络以及其它主机。进而达到了安全隔离的效果。

### **云原生网络功能**

#### （1）云原生网络架构

容器平台包括容器引擎Runtime（如containerd，cri-o等），容器网络接口（CNI，如calico，flannel，contiv，cilium等）和容器存储接口（CSI，如EBS CSI，ceph-csi等）。

![](/assets/compute-hardware-dpu-netcni1.png)

云原生对于网络的需求，既有基础的二三层网络联通，也有四至七层的高级网络功能。二三层的网络主要是实现K8S中的CNI接口，具体如calico，flannel，weave，contiv，cilium等。主要是支持大规模实例，快速弹性伸缩，自 愈合，多集群多活等。四至七层网络功能，主要是服务网格（Service Mesh）。

#### （2）eBPF的硬件加速

eBPF是一项革命性的技术，可以在Linux内核中运行沙盒程序，而无需重新 编译内核或者加载内核模块。在过去几年，eBPF已经成为解决以前依赖于内核更改或者内核模块的问题的标准方法。对比在Kubernetes上Iptables的转发路径， 使用eBPF会简化其中大部分转发步骤，提高内核的数据转发性能。Cilium是一个基于eBPF实现的开源项目，提供和保护使用Linux容器管理平台部署的应用程序服务之间的网络和API连接，以解决容器工作负载的新可伸缩性，安全性和可见性要求。

### RDMA网络功能

#### （1）RDMA网络功能介绍

面对高性能计算、大数据分析和浪涌型IO高并发、低时延应用，现有TCP/IP软硬件架构和应用高CPU消耗的技术特征根本不能满足应用的需求。这主要体现在处理时延过大——数十微秒，多次内存拷贝、中断处理，上下文切换，复杂的TCP/IP协议处理，以及存储转发模式和丢包导致额外的时延。而RDMA通过网络在两个端点的应用软件之间实现Buffer的直接传递，相比TCP/IP，RDMA无需操作系统和协议栈的介入，能够实现端点间的超低时延、超高吞吐量传输，不需要网络数据的处理和搬移耗费过多的资源，无需OS和CPU的介入。RDMA的本质实际上是一种内存读写技术。

![](/assets/compute-hardware-dpu-rdma1.png)

RDMA和TCP/IP网络对比可以看出，RDMA的性能优势主要体现在：

（1）零拷贝——减少数据拷贝次数，由于没有将数据拷贝到内核态并处理数据包头部到过程，传输延迟会显著减少。

（2）Kernel Bypass和协议卸载——不需要内核参与，数据通路中没有繁琐的处理报头逻辑，不仅会使延迟降低，而且也节省了CPU的资源。

#### （2）RDMA硬件卸载方式

原生RDMA是IBTA（InfiniBand Trade Association）在2000年发布的基于InfiniBand的RDMA规范；基于TCP/IP的RDMA称作iWARP，在2007年形成标准；基于Ethernet的RDMA叫做RoCE，在2010年发布协议，基于增强型以太网并 将传输层换成IB传输层实现；在2014年，IBTA发布了RoCEv2，引入IP解决扩展性问题，可以跨二层组网，引入UDP解决ECMP负载分担等问题。

![](/assets/compute-hardware-dpu-rdma2.png)

InfiniBand是一种专为RDMA设计的网络，从硬件级别保证可靠传输。全球HPC高算系统TOP500大效能的超级计算机中有相当多套系统在使用InfiniBand Architecture（IBA）。最早做InfiniBand的厂商是IBM和HP，现在主要是NVIDIA的Mellanox。InfiniBand从L2到L4都需要自己的专有硬件，成本非常高。

iWARP直接将RDMA实现在TCP上，优点就是成本最低，只需要采购支出 iWARP的NIC即可以使用RDMA，缺点是性能不好，因为TCP协议栈本身过于重量级，即使按照iWARP厂商的通用做法将TCP卸载到硬件上实现，也很难超越 IB和RoCE的性能。

RoCE（RDMA over Converged Ethernet）是一个允许在以太网上执行RDMA的网络协议。由于底层使用的以太网帧头，所以支持在以太网基础设施上使用 RDMA。不过需要数据中心交换机DCB技术保证无丢包。相比IB交换机时延，交换机时延要稍高一些。由于只能应用于二层网络，不能跨越IP网段使用，市场应用场景相对受限。

RoCEv2协议构筑于UDP/IPv4或UDP/IPv6协议之上。由于基于IP层，所以可以被路由，将RoCE从以太网广播域扩展到IP可路由。由于UDP数据包不具有保序的特征，所以对于同一条数据流，即相同五元组的数据包要求不得改变顺序。另外，RoCEv2还要利用IP ECN等拥塞控制机制，来保障网络传输无损。RoCEv2也是目前主要的RDMA网络技术，以NVIDIA的Mellanox和Intel为代表的厂商，均支持RoCEv2的硬件卸载能力。

## **存储功能卸载**

可以使用 DPU 来支持数据中心的存储。例如，可以通过将 NVMe 存储设备连接到 DPU 的 PCIe 总线来加速对它们的访问。

DPU 还可实现更好地访问依赖 NVMe-oF 的远程存储设备。DPU 将这些远程存储设备作为标准 NVMe 设备呈现给系统。优化了与远程存储的连接，不再需要特殊的驱动程序来连接这些远程存储设备。

### **NVMe-oF硬件加速**

NVMe over Fabric（又名NVMe-oF）是一个相对较新的协议规范，旨在使用NVMe通过网络结构将主机连接到存储，支持对数据中心的计算和存储进行分解。NVMe-oF协议定义了使用各种通用的传输协议来实现NVMe功能的方式。在NVMe-oF诞生之前，数据存储协议可以分为三种：

* （1）iSCSI：是一种基于IP的存储网络标准，在TCP/IP网络上通过发送 SCSI命令来访问块存储服务。

* （2）光纤通道（Fibre Channel）：是一种高速的数据传输协议，提供有序无损的块数据传输。主要用于关键高可靠要求的业务上。

* （3）SAS（Serial Attached SCSI）：一种点对点串行协议，通过SAS线缆传 输数据。

上述数据存储协议，在当今数据爆发的时代，已经无法满足大数据量的传 输。NVMe-oF的出现，不仅解决了上述协议的性能瓶颈问题，它还允许组织为 高度分布式、高度可用的应用程序实施横向扩展的存储。通过将NVMe协议扩展到SAN设备，NVMe-oF提高了CPU的使用效率，同时提高了服务器和存储应用程序之间的连接速度。

NVMe-oF主要支持三大类Fabric传输选项，分别是FC、RDMA和TCP，其中RDMA支持InfiniBand、RoCEv2和iWARP。

NVMe-oF/FC和第六代FC可以共存于同一基础设施中，避免了数据中心的叉车升级。但是，NVMe-oF/FC不具有软件定义存储的能力。NVMe-oF/RDMA利用了RDMA网络的优势，是理想的Fabric，提供了低延 迟、低抖动和低CPU使用率低传输层协议，可以最大限度利用硬件加速，避免软件协议栈开销。同时，由于RDMA是一种内存读写技术，可以应用在众多场景中，如GPUDirect Storage的应用场景。

NVMe-oF/TCP利用了TCP协议的可靠性传输的特点，以及TCP/IP网络的通用性和良好的互操作性，可以完美的应用于现代数据中心网络。在相对性能要求不是非常高的场景，NVMe-oF/TCP可作为备选。

NVMe支持Host端（Initiator或Client）和Controller端（Target或Server），目 前DPU智能网卡硬件加速的场景中，包括如下四中情况：

* （1）普通智能网卡硬件加速NVMe-oF Initiator。智能网卡支持NVMe-oF/TCP和NVMe-oF/RoCEv2作为Initiator，通过硬件卸载NVMe-oF/TCP或NVMe- oF/RoCEv2，用于计算和存储之间，来达到较高性能。

* （2）支持GPUDirect Storage的智能网卡加速NVMe-oF Initiator和Target。GPUDirect Storage是NVIDIA提出的GPU可以绕过CPU直接访问存储磁盘的技术，RDMA技术是GPUDirect Storage的基础。这类网卡可以通过硬件卸载NVMe- oF/RDMA来实现GPU与远端存储服务的直接访问。常见的如NVMe-oF/RDMA IB和NVMe-oF/RoCEv2。

* （3）智能网卡硬件加速NVMe-oF Target。该场景主要是通过智能网卡提供PCIe Root Complex能力和NVMe-oF Controller端的硬件卸载加速，来实现NVMe存储服务器。如Broadcom Stingray PS1100R是这个场景的代表之一。

* （4）DPU芯片硬件加速NVMe-oF Target。该场景是通过DPU芯片提供多个PCIe Root Complex通道以及多个100Gbps的网卡实现的超大吞吐的存储服务器。Fungible FS1600 12x100Gbps带宽吞吐的存储服务器是这个场景的典型代表。  
  ![](/assets/compute-hardware-dpu-nvmeof1.png)

OpenStack从Rocky版本已经支持了NVMe-oF，通过OpenStack Cinder通过消息在NVMe-oF Target上来创建，查询和删除卷等，OpenStack Nova在主机上通过NVMe-oF Initiator发现NVMe-oF存储设备，并将存储设备信息传递给Hypervisor来实现虚拟机挂载磁盘。另外，OpenStack集成Ceph做块存储和对象存储已经非常成熟，Ceph的后端存储也渐渐的从使用本地磁盘的方式转向远端NVMe存储，这样NVMe-oF为Ceph存储服务提供了容量可伸缩的能力。

### **Virtio-blk硬件加速**

基于virtio的virtio-blk是KVM-Qemu虚拟化生态中的虚拟化块存储的一种实 现方式，利用了virtio共享内存的机制，提供了一种高效的块存储挂载的方法。Guest OS内核通过加载virtio-blk驱动，实现块存储的读写，无需额外的厂家专用驱动。Virtio-blk设备在虚拟机以一个磁盘的方式呈现，是目前应用最广泛的虚拟存储控制器。

![](/assets/compute-hardware-dpu-vpda1.png)

由于virtio机制通过硬件实现加速已经是通用做法，所以利用这个优势，virtio-blk卸载到硬件，已经是必然趋势。在智能网卡中，将virtio-blk到后端映射到如NVMe-oF的远端磁盘上，这样相比较当前virtio-blk的用法，不需要在主机系统中挂载很多的远端NVMe磁盘，由智能网卡直接完成映射，更加安全。

## **安全功能卸载**

### **硬件信任根**

硬件信任根在安全领域是其它安全功能的基础，主要表现如下方面：

* （1）硬件信任根（Root-Of-Trust）：硬件信任根提供更离散的密钥生成算法，并且与主机操作系统相隔离，可以做到硬件防破解。硬件信任根实现私有 密钥存储，可以反克隆和签名。通过硬件信任根认证授权实现访问受控。

* （2）加密解密（Encryption/Decryption）：数据加密解密算法完全卸载到硬件网卡，无需主机CPU资源，效率更高更可靠。可以实现通用加密算法和国 密算法等。

* （3）密钥证书管理（KMS）：密钥证书管理卸载到智能网卡，与主机系统相隔离；支持多种密钥交换算法，如D-H密钥交换等。

* （4）动态数据安全（Secure Data-in-Motion）：利用硬件级加解密算法，对 传输通道上的数据做加解密处理，如IPSec和TLS等。硬件处理可以实现更高吞 吐量。

* （5）静态数据安全（Secure Data-at-Rest）：在存储服务中，永久存盘的数据需要进行加密，防止被窃取，硬件级数据加解密在存储服务中可以提供更高效的数据读取，并保证数据安全。

* （6）流日志和流分析（Flowlog）：流分析和流日志监控，对数据中心流量做精细监控，有效识别，可以及时识别DDoS攻击，并做出响应。

### **安全服务应用**

在安全领域，还有很多的安全功能产品，如NGFW，WAF，IPS/IDS，DDoS防御设备等。随着云和虚拟化技术的发展，越来越多的安全功能产品的实 现方式转为虚拟化方式，并通过云平台来部署管理。这些安全功能产品由于部署在数据中心流量的主要路径上，转发性能对整体网络的吞吐量和时延具有重要的影响。基于X86的软件实现方式，需要大量CPU资源来处理对应的业务逻 辑，性能上的瓶颈已经愈发明显。通过智能网卡对这些安全功能产品做硬件加速，已经是必然趋势。

![](/assets/compute-hardware-dpu-security1.png)

由于安全功能产品对报文处理的深度不同，有些只需要在二至四层处理，有些则需要在七层进行处理，所以在智能网卡的卸载方式上，也存在不同。如NGFW和DDoS等设备，可以通过流表卸载的方式，对流量进行拦截，来加速运行在主机系统中的安全服务应用。如IPS/IDS等，需要对报文内容做深度检测， 则可以通过in-line的方式将数据深度检测功能卸载到智能网卡的CPU上，这时需要智能网卡的CPU具有较强的性能。

