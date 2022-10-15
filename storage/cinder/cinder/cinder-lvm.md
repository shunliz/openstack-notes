OpenStack 中的实例是不能持久化的，cinder服务重启，实例消失。如果需要挂载 volume，需要在 volume 中实现持久化。Cinder提供持久的块存储，目前仅供给虚拟机挂载使用。它并没有实现对块设备的管理和实际服务，而是为后端不同的存储结构提供了统一的接口，不同的块设备服务厂商在 Cinder 中实现其驱动支持以与 OpenStack 进行整合。它通过整合后端多种存储，用API接口为外界提供存储服务。主要核心是对卷的管理，允许都卷、类型和快照进行处理。

Cinder存储分为本地块存储、分布式块存储和SAN存储等多种后端存储类型：  
1. 本地存储： 默认通过LVM支持Linux  
2. SAN存储：  
    （1）通过NFS协议支持NAS存储，比如Netapp  
    （2）通过添加不同厂商的制定driver来为了支持不同类型和型号的商业存储设备，比如EMC,IBM的存储。 在 [https://wiki.openstack.org/wiki/CinderSupportMatrix](https://wiki.openstack.org/wiki/CinderSupportMatrix)可以看到所支持的厂商存储列表。  
3. 分布式系统：支持sheepdog，ceph，和IBM的GPFS等

对于本地存储，cinder-volume 默认使用 LVM 驱动，该驱动当前的实现需要在主机上事先用 LVM 命令创建一个的卷组 , 当该主机接受到创建卷请求的时候，cinder-volume 在该卷组 上创建一个逻辑卷, 并且用 openiscsi 将这个卷当作一个 iscsi tgt 给输出.还可以将若干主机的本地存储用 sheepdog 虚拟成一个共享存储，然后使用 sheepdog 驱动。

