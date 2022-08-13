# BareMetal裸金属

## 简介

## xxxx

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



