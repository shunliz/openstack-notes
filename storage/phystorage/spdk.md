# 什么是SPDK？

        性能开发工具包（SPDK）提供了一组工具和库，用于编写高性能，可伸缩的用户模式存储应用程序。它通过使用一些关键技术实现了高性能：



将所有必需的驱动程序移动到用户空间，这样可以避免系统调用并启用应用程序的零拷贝访问。

轮询硬件用于完成而不是依赖中断，这降低了总延迟和延迟差异。

避免I / O路径中的所有锁定，而是依赖于消息传递。

SPDK的基石是用户空间，轮询模式，异步，无锁NVMe驱动程序。这提供了从用户空间应用程序直接到SSD的零拷贝，高度并行访问。驱动程序被编写为带有单个公共头的C库。有关详细信息，请参阅 7.1 NVMe驱动程序。

SPDK还提供了一个完整的块堆栈作为用户空间库，它执行许多与操作系统中的块堆栈相同的操作。这包括统一不同存储设备之间的接口，排队以处理诸如内存不足或I / O挂起以及逻辑卷管理等情况。有关详细信息，请参阅3.6块设备用户指南。

最后，SPDK提供基于这些组件构建的NVMe-oF，iSCSI和vhost服务器，这些服务器能够通过网络或其他进程提供磁盘。NVMe-oF和iSCSI的标准Linux内核启动器与这些目标以及带有vhost的QEMU互操作。与其他实现相比，这些服务器的CPU效率可高达一个数量级。这些目标可用作如何实现高性能存储目标的示例，或用作生产部署的基础。

# SPDK软件架构概述

      SPDK如何工作？通过结合两种关键技术来实现极高的性能:运行在 user level和使用Poll Mode Drivers\(PMDs,轮询模式驱动程序\)。让我们看看这两个软件工程术语。



      首先，根据定义，在用户级别运行设备驱动程序代码意味着驱动程序不在内核中运行。避免内核上下文切换和中断可以节省大量的处理开销，从而可以将更多的时间花费在真实的数据存储。无论存储算法的复杂性如何（删除重复数据，加密，压缩或普通block存储），更少的无用周期都意味着更好的性能和延迟。这并不是说内核会增加不必要的开销，相反，内核增加了与通用计算用例相关的开销，而这些用例可能不适用于专用存储堆栈。SPDK的指导原则是通过消除每个额外的软件开销来源来提供最低的延迟和最高的效率。



      其次，PMDs更改了I / O的基本模型。在传统的I / O模型中，应用程序先提交读取或写入请求，然后休眠，在I / O完成后等待中断将其唤醒。而PMDs的工作方式有所不同，应用程序先提交读取或写入请求，然后去做其他工作，每隔一段时间检查一次I / O是否已完成，这避免了使用中断的等待时间和开销，并允许应用程序提高I / O效率。在旋转媒介（磁带和HDD）的时代，中断的开销仅占总I / O时间的一小部分，因此极大地提高了系统的效率。但是，随着固态媒体时代的到来，持续引入低延迟的持久性介质，中断开销已成为整个I / O时间的重要部分。更低延迟的介质只会使这一挑战更加明显。系统已经能够每秒处理数百万个I / O，消除数百万个事务的开销，从而迅速节省了多个内核。数据包和数据块被立即分派，等待时间最少化，从而降低了等待时间，提高了等待时间的一致性（减少了波动）并提高了吞吐量。



      SPDK由许多子组件组成，这些子组件相互链接并共享用户级和轮询模式操作的通用元素。创建每个组件都是为了克服创建end-to-end SPDK架构时遇到的特定性能瓶颈。但是，每个组件也可以集成到非SPDK架构中，从而使客户可以利用SPDK的经验和技术来加速自己的软件。

![](/assets/storage-virtual-spdk1.png)**2.1 硬件驱动层**

NVMe驱动：SPDK的基础组件，这个高度优化的无锁驱动程序提供了极好的可扩展性，效率和性能。

Intel® QuickData Technology：也称为Intel® I/O Acceleration Technology（Intel® IOAT，Intel® I / O加速技术），这是内置基于Intel® Xeon®处理器平台中的复制卸载引擎\(copy offload引擎\)。通过提供用户空间访问，减少了DMA数据移动的阈值，从而更好地利用小型的I/O或NTB。

**2.2 后端块设备层**

NVMe over Fabrics \(NVMe-oF\) 启动器:关于NVMe和NVMe-oF的关系，可参考《谈谈关于NVMe和NVMe-oF的那些事》，从程序员的角度来看，本地SPDK NVMe驱动程序和NVMe-oF启动器共享一组通用的API命令，这意味着local/remote复制非常容易启用。

Ceph\* RADOS Block Device \(RBD\)：支持Ceph作为SPDK的后端设备，这可能允许Ceph用作另一个存储层。

Blobstore Block Device：一个由SPDK Blobstore分配的块设备，这是一个虚拟设备，应用于虚机或数据库场景。这些设备享受SPDK基础设施带来的好处，这意味着零锁和极佳的可扩展性能。

Linux\* 异步 I/O \(AIO\): 允许SPDK与HDD之类的内核设备进行交互。

**2.3 存储服务层**

Block device abstraction layer （bdev，块设备抽象层）：这种通用的块设备抽象层是将存储协议连接到各种设备驱动程序和块设备的粘合剂。还为块层中的其他用户功能（RAID，压缩，删除重复数据等）提供了灵活的API。

Blobstore：为SPDK实现高度简化的类似于文件的语义（非POSIX \*）。这可以为数据库，容器，虚拟机（VM）或其他不依赖于POSIX文件系统功能集（例如用户访问控制）的大部分工作负载提供高性能基础。

**2.4 存储协议**

iSCSI target：实现已建立的以太网块通信规范，效率是内核 LIO（linux IO）的两倍，当前版本默认使用内核TCP/IP栈。

NVMe-oF target：实现新的NVMe-oF规范，尽管它取决于RDMA硬件，但NVMe-oF target 可以为每个CPU核心提供高达40gbps的流量。

vhost-scsi target：KVM/QEMU的一项功能，它利用SPDK NVMe驱动程序，使访客虚拟机（Guest VMs）可以更低延迟地访问存储介质，并减少I/O密集型工作负载的总体CPU负载。

# SPDK逻辑架构

        从流程上来看，spdk有数个子构件组成，包括网络前端、处理框架和存储后端。

![](/assets/storage-virtual-spdklogic.png) 前端由DPDK、网卡驱动、用户态网络服务构件组成。DPDK给网卡提供一个高性能的包处理框架；网卡驱动提供一个从网卡到用户态空间的数据快速通道；用户态网络服务则破解TCP/IP包并生成iSCSI命令。 



        处理框架得到包的内容，并将iSCSI命令翻译为SCSI块级命令。不过，在将这些命令送给后端驱动之前，SPDK提供一个API框架以加入用户指定的功能，即spcial sauce（上图绿框中），例如缓存、去冗、数据压缩、加密、RAID和纠删码计算等，诸如这些功能都包含在SPDK中。不过这些功能仅仅是为了帮助我们模拟应用场景，需要经过严格的测试优化才可使用。



        数据到达后端驱动，在这一层中与物理块设备发生交互，即读与写。SPDK包括了几种存储介质的用户态轮询模式驱动：



NVMe设备；

Linux异步IO设备如传统磁盘；

基于块地址的内存应用的内存驱动（如RAMDISKS）；

可以使用Intel I/O加速技术设备；



# SPDK使用评估

        SPDK并不适合所有的存储架构。这里有一些问题可以帮助您确定SPDK组件是否适合您的架构。



1. 存储系统是基于Linux还是FreeBSD \*？



      SPDK主要在Linux上经过测试和支持。FreeBSD和Linux都支持硬件驱动程序。



2. 存储系统是否是英特尔®系统架构的硬件平台？



      SPDK旨在充分利用英特尔®的平台特性，并针对英特尔®芯片和系统进行了测试和调整。



3. 存储系统的性能路径当前是否在用户模式下运行？



      SPDK可以通过在用户空间中运行更多性能路径\(performance path\)来提高性能和效率。通过将应用程序与SPDK功能（例如NVMe-oF target，启动器或Blobstore）组合在一起，整个数据路径可能在用户空间中运行，从而提高了效率。



4. 系统架构能否将无锁PMD纳入其线程模型？



      由于PMD持续在其线程上运行（而不是在未使用时休眠或转让处理器），因此它们具有特定的线程模型要求。



5. 系统当前是否使用DPDK（Data Plane Development Kit，数据平面开发套件）来处理网络数据包工作负载？



      SPDK与DPDK共享原语和编程模型，因此当前使用DPDK的客户可能会发现与SPDK的紧密集成很有用。同样，如果客户使用SPDK，则添加DPDK功能进行网络处理可能会带来巨大的机会。



6. 开发团队是否具有专业知识，可以自己理解问题并进行故障排除？



      英特尔对该参考软件不承担任何支持义务。尽管英特尔和SPDK周围的开源社区将采取商业上合理的努力来调查未经修改的已发布软件的潜在勘误，但在任何情况下，英特尔均不对客户承担任何提供软件维护或支持的义务。



      性能测试中使用的软件和工作负载可能仅针对英特尔微处理器的性能进行了优化。使用特定的计算机系统，组件，软件，操作和功能来测量性能测试（例如SYSmark \*和MobileMark \*）。这些因素的任何变化都可能导致结果变化。



