## 1，vDPA框架应用场景 {#h_537468859_0}

![](/assets/compute-lqk-virtio-vdpa1.png)

vDPA设备的数据面遵循virtio标准但控制面是厂商私有的。上面的图展示了vDPA框架应用于虚机的场景，host kernel中的vDPA框架将来自VM的virtio控制面命令翻译到NIC厂商私有控制面命令，而数据面则从guest 用户态直达物理网卡里的VF。右边下面的图展示了vDPA框架应用于容器的场景，同样的， host kernel中的vDPA框架提供了virtio控制面和厂商私有控制面之间的翻译，virtio数据面也是从容器直达物理网卡里的VF。比较虚机和容器场景，虽然它们都使用了用户态virtio-net-pmd驱动，但控制面所使用的API有所不同：

•虚机场景下，virtio-net-pmd使用PCI MMIO空间的寄存器操作VF设备，再由host上的qemu将其翻译为vDPA内核API\(使用系统调用\)。

•容器场景下， virtio-net-pmd直接调用vDPA内核API\(使用系统调用\)。

# 2，vDPA框架的历史考虑

## 2.1，vDPA DPDK\(host\) framework

vDPA DPDK\(host\)框架最早使用vhost-user接口经DPDK将数据面卸载至DPDK PMD或者硬件。Vhost protocol是一种virtio数据面卸载协议，目前看可以卸载至4个地方：QEMU, DPDK\(称之为vhost-user\)，kernel\(称之为vhost-net\)，hardware\(称之为vhost-vDPA\)。vDPA DPDK\(host\)框架将hardware视作另一种vhost-user后端，这种做法虽然也能work但有很多限制：

•这引入了另一种依赖，即vDPA框架依赖于DPDK框架。

•vDPA DPDK\(host\)框架只提供用户态API，没有考虑数据面卸载至host kernel的场景，因而会损失一些内核功能，比如eBPF的支持。

•DPDK专注于数据面，缺少一些配置和控制硬件的工具。

使用kernel vDPA框架的好处是可以将vDPA硬件当做是与QEMU/DPDK/KERNEL等同的另一种数据面卸载目标组件，而不仅仅是另一种vhost-user后端。这样也能移除vDPA框架对DPDK的依赖，还能同时被用户态驱动和内核态驱动使用，进而能复用内核态原有的virtio 网络/存储驱动。将vDPA框架放在内核态还可以复用已有的sysfs/netlink等已有的标准API。

![](/assets/compute-lqk-virtio-vpda2.png)

## 2.2，VFIO-Mdev

vDPA框架也曾经考虑基于VFIO-Mdev框架实现。 VFIO-Mdev框架也是提供一种软件模拟的控制面，它将来自QEMU模拟的PCI设备的控制面命令翻译为厂商私有的命令。社区最终基于以下几点理由考虑放弃基于VFIO-Mdev而独立实现vDPA框架：

•VFIO-Mdev提供的是比较底层的设备抽象，比如它抽象的是PCI配置空间，MMIO空间，地址映射等。而vDPA提供更高层的virtio设备抽象，比如ring/queue/descriptor。

•VFIO定义了一个IOMMU地址翻译模型，它基于平台IOMMU设备，而很多厂商的设备自己集成了地址翻译组件\(比如mellox\)并不使用平台IOMMU设备，这些厂商的NIC的地址翻译模型并不能很好的适配VFIO的模型。

# 3，vDPA kernel framework architecture

vDPA 内核框架的主要目标是隐藏 vDPA hardware 实现的复杂性，并为内核和用户空间提供一个安全统一的接口来使用。从硬件的角度来看，vDPA 可以由多种不同类型的设备实现。如果 vDPA 通过 PCI 实施，则它可以是物理功能 \(PF\)、虚拟功能 \(VF\) 或其他供应商特定的 PF slices，例如子功能 \(SF\)。vDPA 也可以通过非 PCI 设备或甚至通过软件模拟设备来实现。请注意，每个设备可能都有自己的供应商特定接口和 DMA 模型。如右图所示，vDPA 框架抽象了 vDPA 硬件的通用属性，并利用了现有的 vhost 和 virtio 子系统：

vhost子系统是host内核中的Virtio数据面，它以软件方式模拟了virtio 设备的数据路径。Vhost 子系统有一套成熟的用户空间 API，用于通过 vhost device（一种字符设备）配置内核数据面。内核已经开发了针对不同类型vhost device的各种后端（例如网络或 SCSI）。vDPA 内核框架通过该子系统利用这些 API。

Virtio子系统是一套用于将guest/process连接到virtio设备的内核驱动程序框架。一方面它可以作为driver在guest kernel内控制host模拟的virtio设备。另一方面它也可以作为driver在host kernel里运行，这时virtio 设备和 virtio 驱动程序都在主机内核上运行，例如 XDP（eXpress DataPath）用例。

Vhost和virtio两个子系统基本上可以让”host/guest kernel/user driver”都基于vDPA框架在同一物理vDPA网卡上运行起来。

# 4，vDPA kernel framework是如何工作的？

![](/assets/compute-lqk-virtio-vdpa3.png)

对于用户空间驱动程序，vDPA 框架将呈现一个 vhost 字符设备。这允许用户空间 vhost 驱动程序控制 vDPA 设备，就好像它们是 vhost device一样。各种用户空间应用程序可以通过这种方式连接到vhost device。一个典型的用例是将 vDPA 设备与 QEMU 连接起来，并将其用作运行在guest内部的 virtio 驱动程序的 vhost 后端。可以将vDPA device看做是vhost-net之外的另一种类型的数据面卸载的目标\(vhost-vDPA\)。

对于内核 virtio 驱动程序，vDPA 框架将提供一个 virtio 设备。这允许内核 virtio 驱动程序像控制标准 virtio 设备一样控制 vDPA 设备。各种内核子系统可以连接到 virtio 设备以供用户空间应用程序使用。

vDPA 框架是一个独立的子系统，旨在灵活地实现新的硬件功能。为了支持热迁移，该框架支持通过现有的 vhost API 保存和恢复设备状态。该框架可以很容易地扩展到支持硬件脏页跟踪或软件辅助脏页跟踪。由于内核集成，vDPA 框架还可以轻松扩展支持 SVA和 PASID等技术。

除了door bell寄存器（通知硬件有工作要做）之外，vDPA 框架不允许直接的硬件寄存器映射\(不允许在guest直接操作vDPA硬件寄存器\)。为了支持 vDPA，需要在host和guest之间实现一个用户态的中介层\(如 QEMU\)，以作为virtio控制面的中介。这种方法比设备透传（在 SR-IOV 中使用）更安全，因为它最大限度地减少了来自通用寄存器的攻击\(guest并不直接读写硬件寄存器\)。vDPA 框架也更自然地适合新的设备切片技术，例如scalable IOV 和sub function \(SF\)，这些技术也需要软件的中间模拟层。

# 5，vDPA hardware driver

vDPA 框架抽象了 vDPA 设备使用的通用属性和配置操作。按照此抽象，一个典型的 vDPA 驱动程序需要实现以下功能：

设备探测/移除： 供应商特定的设备探测和移除程序。

中断处理： 供应商或平台特定的中断分配和处理。

vDPA 设备抽象：实现 vDPA 框架所需的操作集，其中大部分只是将 virtio 控制命令翻译为供应商特定设备接口。另外设备抽象还负责将 vDPA 设备注册到框架。

DMA 操作：对于继承了DMA 翻译逻辑的设备，它可以要求框架将 DMA mapping传递给驱动程序以实现供应商特定的 DMA 翻译。

由于数据面卸载到了vDPA 硬件，因此hardware vDPA driver薄而易于实施。用户态 vhost driver或内核 virtio driver通过 vhost ioctls 或 virtio 总线命令配置硬化的数据面。vDPA 框架会将 virtio/vhost 命令转发到hardware vDPA driver，后者以供应商特定方式实施这些命令。vDPA 框架还会将来自 vDPA 硬件的中断投递至到用户态 vhost driver和内核 virtio driver。该框架还将支持door bell和中断直通，以达到设备原生性能。

# 6，vDPA devices

VDPA设备的数据面遵循virtio标准但控制面是厂商特定的。vDPA 设备既可以位于物理硬件上，也可以由软件模拟。vDPA 硬件可能的类型包括：

•PF（Physical Function）： 单个的physical function。

•VF（Virtual Function）： SRIOV设备VF代表了设备的一个虚拟化实例，可以独立地分配给不同的使用者\(host application/guest VM\)。

•VDEV（虚拟设备）： scalable IOV中OS可用一个或多个 ADI（轻量级可分配设备接口，一个物理设备可有多个ADI）创建出一个虚拟设备。

•SF（子功能）： 供应商特定接口，用于将物理功能切片为多个子功能，这些子功能可以作为虚拟设备分配给不同的使用者。

尽管这里讨论了PCIe 硬件，但实现也可以不是 PCIe 硬件。根据完成 DMA 翻译的方式和位置，vDPA 设备分为两种类型：

•基于平台的DMA翻译： 从驱动程序的角度来看，设备所有对内存数据的访问都会使用平台 IOMMU 进行地址翻译。例如PCIE vDPA，其 DMA 请求会打上BDF tag，再经由PCIE 总线级别的IOMMU进行DMA地址翻译和安全保护检查。

•基于设备的 DMA翻译： 设备通过自己的逻辑实现 DMA 隔离和保护, 如片上实现的 IOMMU。请注意，此类片上 IOMMU 可以与平台 IOMMU 合作进行两阶段 DMA 转换，如使用片上 IOMMU 作为第 1 级 DMA 翻译再使用平台 IOMMU 作为第2级翻译。

为了隐藏上述vDPA 设备类型的差异和复杂性，我们需要一个抽象设备框架向上层呈现通用的virtio/vhost 设备。通过vDPA框架和 vhost/virtio子系统，我们可以让内核态virtio驱动和用户态vhost驱动认为它们在控制一个virtio/vhost设备而实际上是一个vDPA设备。

# 7，vDPA bus

vDPA框架实现了一个vDPA软总线，可以将vDPA device和vDPA bus driver都挂在其上。vDPA总线抽象了vDPA device的一些通用属性使得vDPA bus driver可以将vDPA device连接至其它内核子系统。以下是vDPA bus定义的vDPA device和vDPA bus driver间的一套抽象接口\(vdpa\_config\_ops\):

•virtio specific operations：用于控制标准virtio设备，这包括：get/set设备配置空间，get/set virtqueue状态，virtio特性协商。

•Interrupt operations：允许vDPA bus driver为virtqueue/configuration注册中断回调函数，提供helper库函数支持中断转发 \(比如转发intel posted interrupt\)。

•Doorbell operations：允许vDPA bus driver注册virtqueue被kick时的回调函数，报告door bell的内存地址以将其映射至用户态空间。

•Migration operations：包括get/set device/virtqueue状态的接口，收集被硬件修改的脏页的接口。

•DMA mapping operations ：当vDPA设备有其自己的DMA翻译逻辑时，需要一个接口在设备上配置DMA映射表。

vdpa\_config\_ops被定义成一套大而全的接口以适应各种设备和驱动。vDPA bus driver可以选择性的只使用其中一部分，比如仅在VM热迁移时使用脏页跟踪的API。通过抽象出vDPA bus和vdpa\_config\_ops操作集，底层硬件的差异和复杂性被向上屏蔽。 vDPA bus drivers 和vDPA device可以基于统一的vdpa\_config\_ops总线操作集相互协作，而不需要了解彼此的实现细节。

# 8，vDPA physical device driver

vDPA物理设备驱动在vDPA bus的下方，它向vDPA bus呈现出一个虚拟的vDPA设备，将vDPA总线操作集转换为厂商私有接口之后跟物理设备通信。这里vDPA物理设备驱动承担了2个角色：从硬件的角度看它是vDPA物理设备的驱动，从vDPA总线角度看它是一个vDPA设备。

vDPA物理设备驱动需要完成以下事情：

•vDPA device probing/removing：通过厂商私有方式或者平台方式探测出物理的vDPA硬件设备，并向vDPA总线注册vDPA设备。

•vDPA device management：vDPA物理设备可能是类似于SRIOV VF一样的虚拟设备，它需要与PF驱动配合管理vDPA物理设备的生命周期。

•Interrupt processing：分配并处理vDPA物理设备中断，并负责将每个virtqueue的中断上报至vDPA bus，而vDPA bus则将中断重定向给KVM进而注入虚机。

•Device abstraction：实现vdpa\_config\_ops总线操作集，将抽象的vDPA bus操作翻译为特定的平台或厂商的物理设备访问方式。

•Perform DMA mapping：对于自己实现DMA翻译的vDPA物理设备来说， vDPA物理设备驱动需要接收来自vDPA总线的DMA映射请求，并将其翻译为厂商内部的数据结构以便能配置到vDPA物理设备中。 vDPA物理设备驱动还需要向vDPS bus提供一个DMA设备以执行DMA/IOMMU相关的操作，比如IOMMU设备探测，建立DMA映射。

vDPA物理设备驱动使用vdpa\_config\_ops与vDPA总线通信，它是vDPA总线和厂商私有接口之间的一个中介。vDPA bus driver只需看到一个统一的vDPA设备抽象，并使用同一套vdpa\_config\_ops接口跟各种vDPA物理设备通信。

# 9，vDPA bus drivers

vDPA bus driver用于将 vDPA 总线连接到位于 vDPA 内核框架顶部的 vhost 和 virtio 子系统。请注意，这与用于将 vDPA 总线连接到 vDPA 物理设备的 vDPA physical device driver不同。 目前有两种vDPA bus driver：

vhost-vDPA bus driver – 将 vDPA 总线连接到 vhost 子系统，并向用户空间呈现 vhost 字符设备。这对于期望数据面完全绕过内核的情况很有用。用户空间驱动程序可以通过 vhost ioctls 控制 vDPA 设备，就像 vhost 设备一样。一个典型的用例是将IO访问直通到用户态\(或VM\)。

virtio-vDPA bus driver - 将 vDPA 总线桥接到 virtio 总线，并在那里呈现出一个virtio 接口设备，进而可以被network/block/crypto等各种内核子系统使用。不使用 vhost 用户空间 API 的应用程序可以继续使用网络/块/其他内核子系统提供的socket/file等API。

上图显示了不同的 vDPA bus driver及其连接。通过在 vDPA 内核框架中支持不同的 vDPA bus driver，vDPA 获得了灵活性，因为用例不限于使用vhost ioctl进行直接的用户态I/O。大多数内核子系统都可以与 vDPA 设备一起使用，并且可以开发新的 vDPA bus driver来支持新平台和新硬件。

vDPA提供了一套基于sysfs的管理接口来对vDPA device/driver进行管理。这包括命名 vDPA 设备以及为设备选择需绑定的 vDPA bus driver。

•我们可以使用以下命令来根据PCI名称获得vDPA设备名称：\# ls /sys/bus/pci/devices/0000\:07\:00.2/ \| grep vdpa

•也可以通过命令 \# ls /sys/bus/vdpa/devices/ 获取已有的vDPA设备。

•另一方面我们也可以给vDPA设备切换它所绑定的driver：

\# ls -l /sys/bus/vdpa/devices/vdpa0/driver                 \#首先查看设备当前绑定的driver

\# echo vdpa0 &gt; /sys/bus/vdpa/drivers/vhost\_vdpa/unbind     \#解除设备绑定的原有driver

\# echo vdpa0 &gt; /sys/bus/vdpa/drivers/virtio\_vdpa/bind      \#将设备绑定至新的driver

# 10，vhost-vDPA bus driver

Vhost历史上作为一个内核态实现的virtio数据面，它向用户态暴露了成熟的接口\(通过vhost字符设备\)用于qemu配置内核中的数据面。如果我们将vhost-net视作一种virtio数据面卸载的方式，那么vhost-VDPA只不过是另一种virtio数据面卸载的方式，它复用原有的vhost uAPI\(用户态API\)将virtio数据面卸载至vDPA物理硬件而不是卸载至内核。在这种方式下vhost-vDPA bus driver作为vhost uAPI和vDPA vdpa\_config\_ops之间的中介，它向用户态vhost driver呈现出一个vhost device以使用原有vhost uAPI配置数据面。

vhost-vDPA bus driver的职责是将vhost uAPIs转换为vDPA vdpa\_config\_ops操作。在vhost子系统看来， vDPA bus driver只是另一种支持vhost uAPI的vhost device\(vhost-vDPA 加上 之前就支持的vhost-net\)。一旦一个vDPA设备绑定至vhost-vDPA bus driver, 就会创建一个字符设备/dev/vhost-vdpa-x，这样用户态驱动就可以使用vhost uAPI来操作这个设备了。 vhost-vDPA复用了已有的vhost uAPI：

•Setting device owner \(ioctls\)：每个vhost-vdpa设备只能有一个应用进程作为它的owner，它会拒绝其他进程修改其内存映射。

•Virtqueue configurations \(ioctls\)：设置每个virtqueue的基地址，队列深度，set/get队列的CI/PI指针。 vhost-vDPA bus driver需要将这些操作转换为对vDPA vdpa\_config\_ops操作的调用，进而最终到达vDPA设备。

•Setting up eventfd for virtqueue interrupts and kicks \(ioctls\)：vhost使用eventfd将kick事件或virtqueue中断投递至用户态进程。vhost-vDPA bus driver需要将这些操作转换为对vDPA vdpa\_config\_ops操作的调用，进而最终到达vDPA设备。

•Populating device IOTLB：IOTLB消息用于用户态vhost驱动通知vhost-vdpa设备建立新的IOVA到VA的映射, 包含了VHOST\_IOTLB\_UPDATE\(建立映射\)， VHOST\_IOTLB\_INVALIDATE\(撤销映射\)， VHOST\_IOTLB\_MISS\(类似page fault, 设备找不到映射时主动要求驱动建立映射\)三种消息。

传统的vhost ioctl用于配置内核态的virtio数据面，但这对于vDPA设备来说是不够的，因为除了数据面需要配置，我们同样需要配置vDPA设备的控制面，因此需要增加一些ioctl API：

•Get virtio device ID：device ID用于在用户态找到匹配的驱动。

•Set/Get device status：控制设备的start/stop状态。

•Set/get device config space：读写设备配置空间。

•Config device interrupt support：通过eventfd将设备配置变动产生的中断投递至用户态。

•Doorbell mapping：将virtqueue的door bell直接映射至用户态或VM，减少vm-exit，减少使用类似eventfd机制所带来的开销。

通过以上扩展， vhost-vDPA bus driver提供了vDPA设备的完全抽象。用户态可以像控制vhost设备一样控制vhost-vdpa设备。在收到用户态的请求命令后， vhost-vDPA bus driver可以选择自己完成请求或者将请求转换为vDPA总线操作并转发至底层vDPA设备。

# 11，Memory \(DMA\) mapping with vhost-vDPA

IOTLB是vhost-vdpa设备的DMA抽象，它在用户态可见。用户态驱动需要显式的在设备上建立或撤销IOVAàVA的映射。IOVA可能是：

•Guest Physical Address \(GPA\) ：vhost-vdpa作为VM的virtio后端，且VM内没有vIOMMU时，vDPA物理设备使用GPA访问内存。

•Guest IO Virtual Address \(GIOVA\)：vhost-vdpa作为VM的virtio后端且VM内有vIOMMU时，vDPA物理设备使用VM内应用进程定义的GIOVA访问内存。

•Virtual Address \(VA\)： vhost-vdpa将设备IO空间映射至host用户态后，用户态写入硬件的地址是VA地址，所以vDPA物理设备使用VA访问内存。

vhost-vdpa如何处理IOTLB请求取决于使用哪种DMA翻译硬件，它可能是platform IOMMU或设备片上IOMMU。

## 11.1，DMA translation on platform IOMMU

![](/assets/compute-lqk-virtio-vdpa4.png)

当使用platform IOMMU做DMA翻译时，DMA映射的处理流程如右图所示。 vhost-vDPA bus driver 通过检查vDPA设备是否实现了vDPA总线操作集中的DMA config ops来判断vDPA设备是使用platform IOMMU还是on chip IOMMU，没有实现DMA config ops 意味着使用platform IOMMU。vhost-vDPA bus driver会在probe探测vDPA设备时进一步探测设备所关联的platform IOMMU。



我们通过vDPA创建时关联的物理DMA设备\(在右图中指的是PCI SRIOV VF\)来找到关联的platform IOMMU。 vhost-vDPA bus driver会为vhost-vDPA的owner进程分配一个唯一的IOMMU domain以便隔离其它进程发起的DMA操作。自然，删除vhost-vDPA设备时也需要删除这个IOMMU domain。 vhost-vDPA用这个IOMMU domain作为handle在platform IOMMU中创建或删除DMA映射。一旦IOMMU domain建立，则vhost-vDPA device就可以通过IOTLB uAPI接收来自用户态的DMA映射请求了：



•当用户态希望建立或撤销IOVAàVA的DMA映射时向 vhost-vDPA字符设备写入vhost IOTLB消息。



•vhost-vDPA设备接收到消息，将里面的VA转换为PA。注意连续的VA可能对应非连续的PA，所以这里可能会变成多个IOVAàPA的映射。



•vhost-vDPA设备会锁住对应的物理页，然后将IOVAàPA映射记录在自己维护的IOTLB里。



•vhost-vDPA设备调用platform IOMMU API在IOMMU domain内建立IOVAàPA映射。这是一套统一的IOMMU API， 用于隔离用户态发起的DMA。



•IOMMU domain再请求IOMMU driver执行vendor/platform特定的操作以在platform IOMMU硬件中建立/撤销DMA映射。



vDPA物理硬件使用IOVA发起DMA操作，请求会经过platform IOMMU，在那里IOVA会被验证并翻译为PA地址并进行真正的DMA。



## 11.2，DMA translation on chip IOMMU

![](/assets/compute-lqk-virtio-vdpa5.png)

vhost-vDPA bus driver检查vDPA设备提供vDPA总线操作集，若实现了DMA config ops则认为vDPA设备自己实现了DMA翻译。当DMA翻译由vDPA设备自己完成时，来自用户态的地址映射请求会被vhost-vdpa bus driver经vDPA bus转发至底层的vDPA physical device driver。



vhost-vDPA设备接收到消息，将里面的VA转换为PA。注意连续的VA可能对应非连续的PA，所以这里可能会变成多个IOVA-&gt;PA的映射。vhost-vDPA设备会锁住对应的物理页，然后将IOVA-&gt;PA映射记录在自己维护的IOTLB里。vhost-vDPA设备调用DMA config ops将映射请求转发至底层vDPA physical device driver。底层vDPA physical device driver使用厂商私有方式去配置vDPA物理设备上的IOMMU。在需要和platform IOMMU协作的情况下， 比如设备内置的IOMMU执行第一级IOVA-&gt;IA\(intermediate address\)映射，而platform IOMMU执行第二级IA-&gt;PA映射时，vDPA physical device driver也会调用platform IOMMU接口配置翻译页表。



vDPA物理硬件使用IOVA发起DMA操作，请求首先经过on-chip IOMMU翻译为IA，再会经过platform IOMMU翻译为PA地址并进行真正的DMA。通过IOTLB抽象，IOMMU domain和vDPA physical device driver的协作，vhost-vDPA设备向用户态呈现了一个统一的硬件无关的vhost设备，用户态程序可以以一致接口形式去配置vhost device。



# 12，virtio-vDPAbus driver





Virtio-vdpa bus driver向上呈现一个virtio设备从而可以复用内核里的virtio驱动。 Virtio-vdpa bus driver的目标是内核中的virtio驱动像控制一个常规的virtio设备一样去控制模拟出来的vDPA设备，且不需要对virtio driver做任何修改。 Virtio-vdpa bus driver提供了一个内核态的virtio数据面。Virtio-vDPA bus driver在vDPA设备基础之上模拟出virtio设备：



•实现了一个新的vDPA传输层\(对比之前的PCI\)将virtio bus的操作翻译为vDPA bus的操作。



•探测vDPA设备并向virtio bus注册virtio设备。



•将设备中断转发至virtio设备。



Virtio-vDPA bus driver实际是virtio bus和vDPA bus之间的一个中介层：在vDPA bus看来它是vDPA bus的一个driver，vDPA bus并不往上直接看到virtio bus/device；在virtio bus看来它是virtio bus的一个device，virtio bus并不往下直接看到vDPA bus/device；在此抽象之下，基于vDPA device的virtio device能够被virtio driver探测到，进程能使用socket uAPI在virtio网络设备上收发包。



# 13，Memory \(DMA\) mapping with virtio-vDPA driver

![](/assets/compute-lqk-virtio-vdpa7.png)

内核态的驱动使用platform IOMMU做地址翻译时统一使用一个DMA domain，这个DMA domain在系统boot up期间建立。virtio-vDPA bus driver只需要处理DMA映射的建立和撤销而无需关心IOMMU domain的建立。



Platform IOMMU API在建立DMA映射时需要一个device handle，这里virtio-vDPA bus driver向virtio bus注册device时会指定底层vDPA physical device \(parent device，一般是一个PCI设备\)的handle作为virtio device的handle。Platform IOMMU根据handle找到parent device然后根据其所连接到的硬件platform IOMMU将映射请求转发至对应的platform IOMMU driver。



vDPA物理硬件使用IOVA发起DMA操作，请求会经过platform IOMMU，在那里IOVA会被验证并翻译为PA地址并进行真正的DMA。



