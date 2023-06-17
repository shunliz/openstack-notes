# BareMetal裸金属 {#zhou-shunli}

## 简介

裸金属特性是一种将物理设备作为资源提供给租户的云计算服务，租户通过该服务可申请、管理和配置相应的物理设备资源。这种物理设备是未安装操作系统的服务器，又称为裸金属服务器，简称裸金属。

在Openstack中，由Ironic这个组件来提供裸金属的部署和管理。Ironic需要与Keystone、Nova、Neutron、Cinder以及Swift进行交互，像Nova创建虚拟机一样，需要对应的认证服务、网络服务、块存储服务、对象存储服务等。

对Nova而言，通过Ironic部署物理机，和部署虚拟机的调用流程是一样的，都是通过Nova的接口来执行创建实例。不同之处在于，调度裸金属时所用的套餐，与创建虚机的套餐不同；裸金属节点所属的nova-compute驱动与虚机要用的nova-compute驱动不一样；虚拟机底层驱动采用的是虚拟化技术，而物理机采用的是PXE和IPMI技术。以下是Ironic与Nova，Neutron，Glance，Cinder，Swift等组件交互的逻辑架构图。

![](/assets/network_ironic_intro.png)

**裸金属服务的部署阶段**

**部署阶段**

部署阶段也称为Provision/Deploy阶段，租户向云平台申请裸金属资源，云平台为租户分配资源，并为裸金属加载系统镜像。

1）在deploy阶段，ironic-conductor将裸金属设置为PXE引导安装模式，通知neutron准备好TFTP配置，然后启动裸金属。裸金属上电后学习到DHCP报文，根据DHCP报文的TFTP Server地址，向Ironic节点获取kernel及ramdisk镜像并启动，这个ramdisk里包含了ironic-python-agent；在实际使用中，iPXE会比PXE下载镜像的速度更快，推荐使用iPXE更好。

2）启动后，ironic-python-agent会和Ironic控制节点互通，连接到Ironic控制节点的http server，获取完整的用户系统镜像。

**运行阶段        
**

也称为Tenant阶段，裸金属服务器启动系统镜像，业务开始运行

\(1\) 待部署阶段的镜像获取完成，Ironic通知裸金属重启，则进入租户网络运行。

## 裸机三种网络模式

[https://www.h3c.com/cn/d\_202012/1370744\_30005\_0.htm](https://www.h3c.com/cn/d_202012/1370744_30005_0.htm)

[https://docs.openstack.org/ironic/latest/](https://docs.openstack.org/ironic/latest/)

[http://kimizhang.com/neutron-l2-gateway-hp-5930-switch-ovsdb-integration/](http://kimizhang.com/neutron-l2-gateway-hp-5930-switch-ovsdb-integration/)

### **flat网络模式**

![](/assets/network-ironic-flatnet.png)1\) ironic控制节点所在的管理网络（management network）要和裸机的ipmi网络互通，因为要对裸机做开关机，设置bios启动项等操作；

2\) 裸机的deploy网络及tenant网络使用的是同一个flat网络（图中的external network），无需进行切换，且该网络要与管理网络互通；

3\) deploy阶段分配一个external ip，部署结束后的租户网络依然是使用这个ip。

### **vlan网络模式**

![](/assets/network_ironic_vlannet.png)\(1\) 使用直通网络部署，需要在neutron里加入networking-generic-switch这个插件，

\(2\) Ironic服务在部署阶段，会通知插件将裸金属所接的交换机端口配置为使用deploy vlan\(蓝色实线\)

\(3\) 在运行阶段，插件将裸金属所接的交换机端口配置为使用tenant vlan（橘色实线）

\(4\) 部署阶段和运行阶段也可以使用同一个vlan，那么会分配一个相同的IP，无需进行切换。

### vxlan网络

\(1\) 安装networking-l2gw后，在neutron里配置并启动neutron-l2gw-agent

\(2\) 裸金属连接的交换机开启ovsdb的功能。

\(3\) 在neutron里创建l2gw及l2gw connection, neutron会调用l2gw对应的plugin去建立控制节点到交换机，以及计算节点到交换机的隧道，从而来打通虚机到裸机的vpc网络。

![](/assets/network_ironic_vxlannetphy.png)\(4\) 在Ironic控制节点上需要创建一个部署网络里的虚拟网卡，通过这个虚拟网卡，裸金属可以与Ironic控制节点上的ironic服务/tftp server/http server互通，从而获取到部署镜像及用户系统镜像。

![](/assets/network_ironic_vxlannetlp.png)\(5\) 计算节点与交换机建立的隧道信息，以及交换机上学习到的虚机MAC地址信息可以通过执行ovsdb-client dump --pretty tcp:

&lt;交换机ip&gt;:6632 看到![](/assets/network_ironic_vxlannetsnap1.png)![](/assets/network_ironic_vxlansnap2.png)

## 网络切换

### inspect阶段

硬件信息收集过程。调用交换机，修改裸机对应的端口，使其和inspect网络连通。裸机通过网络引导，加载ramdisk，收集硬件信息，通过三层回调到管理网，录入硬件信息，并在neutron中的pxe子网中创建对应mac地址的端口。

### ![](/_images/ironic_netswitch_inspect.png)

### Deploy

部署阶段。调用交换机，修改裸机对应的端口，使其和neutron的PXE网络连通。调用IPIM重启裸机。裸机通过网络引导，加载ramdisk，监听iscsi。管理节点通过iscsi协议将裸机的硬盘挂载到本地，并将对应的系统镜像dd到磁盘中。修改裸机的引导方式为硬盘启动。在内网段中创建网络端口。

![](/assets/ironic_netswitch_deploy.png)

### Active

调用交换机，修改裸机对应的端口，使其和neutron的内网（租户网）连通。裸机从硬盘引导，通过neutron分配到内网ip地址

![](/assets/ironic_netswitch_active.png)

# 裸金属优点

1.2.1 安全方面

        裸金属服务器具有安全物理隔离的特性，裸金属服务器与其他租户物理隔离。



        对安全性要求比较高的用户，例如金融类用户，他们对服务器的安全合规是有硬性要求的，裸金属服务器具有物理机级别的隔离。



1.2.2 性能方面

        裸金属资源完全独占，完全没有性能损耗，能够胜任高 IO 应用、高性能计算等业务，例如海量数据采集和挖掘，高性能数据库，大型在线游戏等。



        特别的，裸金属服务器还可以支持虚拟化，用户可以在裸金属上搭建自己的虚拟化平台，打造独占的私有云或容器云，实现「在公有云上搭建专有云」这样灵活的架构。



1.2.3 弹性和自动化

        除了裸金属的固有特性，裸金属云完全继承了虚拟化云服务器的 云 特性，例如一键自动化快速部署交付、弹性伸缩等，并且整个过程都是自动化管理，无需人工参与即可自动完成镜像安装、网络配置、超级云硬盘挂载等。



        唯一的差距在于相对于虚机和容器的秒级响应，裸金属是分钟级别的响应。



1.2.4 带外批量运维

        支持在线批量管理物理服务器生命周期，如开关机、重启、挂卷、删除、监控、远程登录、状态查询等。



1.2.5 VPC网络+自定义网络

        支持多种方式与虚拟机互联，可以建立虚拟私有网络（VPC），也可以通过自定义网络，以满足不同网络安全需求。VPC网络构建出一个安全隔离的网络环境，可以灵活自定义IP地址范围、配置路由表和网关、 创建子网等，实现弹性IP、弹性带宽、共享带宽等功能。



1.2.6 兼容其它云产品

        裸金属作为云中居民，可以和其它云产品如云主机、云网络、云存储、云数据库直接打通，方便业务使用，构建更加灵活的整体架构和方案。



1.2.7 镜像与备份

        支持云主机镜像创建裸金属主机，支持裸金属主机创建镜像、克隆，支持裸金属主机镜像备份以及恢复。支持配置超高速本地NVMe SSD硬盘及超高速超级云硬盘，并采用大规模分布式存储系统，数据多副本冗余，永不丢失。



1.2.8 管理节点提供部署服务

        小规模部署服务如DHCP服务、TFTP服务、PXE服务等可以由管理节点提供，而不需要单独配置一台部署服务器，造成资源浪费；当然，达到一定规模，单独配置部署服务器十分有必要的，可以减轻管理节点压力，专机专用。



1.2.9 细分场景多种实例

        针对站群、数据库场景、大数据场景、高性能计算和异构计算场景等领域，提供定制化解决方案。



