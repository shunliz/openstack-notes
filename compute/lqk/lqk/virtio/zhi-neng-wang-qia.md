BlueFiled DPU网卡可以从cpu上卸载关键的网络、存储和安全业务，使得企业能够将其IT基础设施转变为最先进的数据中心，单个DPU可以提供相当于125个cpu内核所提供的数据中心服务，由此释放宝贵的cpu内核.

**bluefiled 架构**

![](/assets/compute-lqk-virtio-snic1bf.png)BlueField架构是几个标准现成组件的组合，即Arm AArch64处理器和ConnectX-5（用于BlueField）、ConnectX-6 Dx（用于BlueField-2）或网络控制器，每个组件都有自己丰富的软件生态系统。因此，BlueField中几乎所有可见的软件接口都来自各个组件的现有标准接口。

![](/assets/compute-lqk-virtio-snicbf2.png)Arm相关接口（包括与引导程序、PCIe连接和加密操作加速相关的接口）是标准的Linux-on-Arm接口。这些接口由NVIDIA提供的驱动程序和固件代码实现，这些驱动程序和代码是BlueField软件的一部分，交付给各个开源项目，如Linux。ConnectX网络控制器相关接口（包括用于以太网和InfiniBand连接、RDMA和RoCE以及存储和网络操作加速的接口）与支持ConnectX独立网络控制器卡的接口相同。这些接口利用MLNX\_OFED软件和基于InfiniBand的接口来支持软件。

**系统连接**

![](/assets/compute-lqk-virtio-snicbf3.png)

**ovs全卸载方案**

下图可以看出ovs全卸载到物理网卡上，并在arm操作系统中运行ovs以及相关的程序，通过流表的offload，将流表卸载到固件中，通过固件的eswitch类似于flowcache的过程将代码进行流表匹配，并通过固件转发到host主机上的其他应用中（虚拟机）；

![](/assets/compute-lqk-virtio-snic4bf.png)报文路径,慢路劲需要经过arm核上的ovs进行报文匹配，再发送到host主机中，快路径的报文直接通过固件进行报文转发。 

![](/assets/compute-lqk-virtio-snic6bf.png)

**IPsec全卸载方案**

IPsec完全卸载将IPsec加密和IPsec封装卸载到硬件。通过上行链路netdev在Arm上配置IPsec完全卸载。下图说明了硬件中的IPsec完全卸载操作。

  
![](/assets/compute-lqk-virtio-snic7bf.png)**基于vxlan通道的全卸载**



![](/assets/compute-lqk-virtio-snic8bf.png)![](/assets/compute-lqk-virtio-snic9bf.png)

