![](/assets/compute-lqk-virtio-vdpa1.png)

vDPA设备的数据面遵循virtio标准但控制面是厂商私有的。上面的图展示了vDPA框架应用于虚机的场景，host kernel中的vDPA框架将来自VM的virtio控制面命令翻译到NIC厂商私有控制面命令，而数据面则从guest 用户态直达物理网卡里的VF。右边下面的图展示了vDPA框架应用于容器的场景，同样的， host kernel中的vDPA框架提供了virtio控制面和厂商私有控制面之间的翻译，virtio数据面也是从容器直达物理网卡里的VF。比较虚机和容器场景，虽然它们都使用了用户态virtio-net-pmd驱动，但控制面所使用的API有所不同：

•虚机场景下，virtio-net-pmd使用PCI MMIO空间的寄存器操作VF设备，再由host上的qemu将其翻译为vDPA内核API\(使用系统调用\)。

•容器场景下， virtio-net-pmd直接调用vDPA内核API\(使用系统调用\)。

2，vDPA框架的历史考虑

2.1，vDPA DPDK\(host\) framework



vDPA DPDK\(host\)框架最早使用vhost-user接口经DPDK将数据面卸载至DPDK PMD或者硬件。Vhost protocol是一种virtio数据面卸载协议，目前看可以卸载至4个地方：QEMU, DPDK\(称之为vhost-user\)，kernel\(称之为vhost-net\)，hardware\(称之为vhost-vDPA\)。vDPA DPDK\(host\)框架将hardware视作另一种vhost-user后端，这种做法虽然也能work但有很多限制：



•这引入了另一种依赖，即vDPA框架依赖于DPDK框架。



•vDPA DPDK\(host\)框架只提供用户态API，没有考虑数据面卸载至host kernel的场景，因而会损失一些内核功能，比如eBPF的支持。



•DPDK专注于数据面，缺少一些配置和控制硬件的工具。



使用kernel vDPA框架的好处是可以将vDPA硬件当做是与QEMU/DPDK/KERNEL等同的另一种数据面卸载目标组件，而不仅仅是另一种vhost-user后端。这样也能移除vDPA框架对DPDK的依赖，还能同时被用户态驱动和内核态驱动使用，进而能复用内核态原有的virtio 网络/存储驱动。将vDPA框架放在内核态还可以复用已有的sysfs/netlink等已有的标准API。



