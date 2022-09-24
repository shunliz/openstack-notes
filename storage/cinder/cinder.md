# cinder简介

Cinder提供持久的块存储，目前仅供给虚拟机挂载使用。它并没有实现对块设备的管理和实际服务，而是为后端不同的存储结构提供了统一的接口，不同的块设备服务厂商在 Cinder 中实现其驱动，以与 OpenStack 进行整合。它通过整合后端多种存储，用API接口为外界提供存储服务。

Cinder存储分为本地块存储、分布式块存储和SAN存储等多种后端存储类型：  
1\)本地存储： 默认使用lvm。cinder volume 将该服务所在的节点变为存储节点，将节点上面的 volume group 作为共享存储池暴露给计算节点。  
2\)SAN存储：  
（1）通过NFS协议支持NAS存储，比如Netapp  
（2）通过添加不同厂商的制定driver来为了支持不同类型和型号的商业存储设备，比如EMC,IBM的存储  
3\) 分布式存储：支持sheepdog，ceph，和IBM的GPFS等  
对于本地存储，cinder-volume 默认使用 LVM 驱动，该驱动当前的实现需要在主机上事先用 LVM 命令创建一个卷组 , 当该主机接受到创建卷请求的时候，cinder-volume 在该卷组 上创建一个逻辑卷, 并且用 openiscsi 将这个卷当作一个 iscsi tgt 给输出.还可以将若干主机的本地存储用 sheepdog 虚拟成一个共享存储，然后使用 sheepdog 驱动。

# 2、Cinder LVM配置

在cinder配置文件中，默认的backend lvmdriver是通过LVM来使用某个cinder volume服务所在的服务器的本地存储空间：

cinder-volume服务所在的节点，cinder.conf配置文件内容样式如下：

\[lvmdriver-1\]

volume\_group = stack-volumes-lvmdriver-1  
volume\_driver = cinder.volume.drivers.lvm.LVMISCSIDriver  
volume\_backend\_name = lvmdriver-1

参数详解：  
volume\_group 指定Cinder使用的 volume group即存储节点上的vg。在devstack默认安装时其名称是stack-volumes-lvmdriver-1；在实际部署cinder的时候其默认名称是cinder-volumes。  
volume\_driver 指定driver类型. lvm支持两种传输协议： iSCSI and iSER。  
iSCSI的话，将其值设为 cinder.volume.drivers.lvm.LVMISCSIDriver；  
iSER的话，将其值设为 cinder.volume.drivers.lvm.LVMISERDriver

volume\_backend\_name 指定backend name。当有多个 volume backend 时，需要创建 volume type，它会绑定一个或者多个backend。用户在创建 volume 时需要选择某个 volume type，间接相当于选择了某个 volume backend。

注意事项：

1\)除了在启动的时候读取该配置文件以外，cinder-volume服务不实时监控该文件。因此在修改该文件后你需要重启cinder-volume 服务。  
也就是说cinder-volume服务只是在启动的时候，读取一次该配置文件，剩下的就不在读取了，因此在修改完配置文件后，需要在重启启动一下cinder-volume服务

2\)只有一个backend的时候，除了配置volume group外，不需要添加别的配置信息，创建volume的时候也不需要选择volume type。当有多个backend的时候，你需要使用volume-type来将volume创建到指定的backend中。一个volume-type可以有几个backend，这时候 the capacity scheduler 会自动选择合适的backend来创建volume。如果定义了volume type，但是cinder.conf中没有定义volume backend，那么cinder scheduler将找不到有效的host来创建volume了。

3\)一个存储节点上可以提供多种类型的存储服务，例如，本地存储、ceph存储等，对于本地存储类型来说，一个vg就是一个存储后端backend，一个存储类型type，可以对应多个backend,创建卷时，指定存储类型，系统会自动选择一个有效backend后端进行真实卷的创建。部署的时候，可以这样划分，相同类型的存储后端，归类为一个存储类型，这样可以通过卷类型来区分不同的存储类型。也可以一个存储类型type对应一个存储后端

# 3、实战操作

Cinder存储节点部署,部署在a主机  
1\)安装lvm2软件包  
yum install lvm2 -y

2\)启动LVM的metadata服务并且设置该服务随系统启动  
systemctl enable lvm2-lvmetad.service  
systemctl start lvm2-lvmetad.service

3\)创建LVM 物理卷 /dev/sdb  
pvcreate /dev/sdb

4\)创建 LVM 卷组 cinder-volumes  
vgcreate cinder-volumes /dev/sdb

5\)编辑/etc/lvm/lvm.conf文件并完成下面的操作：  
在devices部分，添加一个过滤器，只接受/dev/sdb设备，拒绝其他所有设备  
devices {  
filter = \[ "a/sdb/", "r/.\*/"\]  
提示：每个过滤器组中的元素都以 a 开头，即为 accept，或以 r 开头，即为\*\*reject\*\*，并且包括一个设备名称的正则表达式规则。过滤器组必须以 r/.\*/ 结束，过滤所有保留设备

6\)安装cinder组件软件包  
yum install openstack-cinder targetcli python-keystone -y

7\)在cinder-volume服务所在的节点修改cinder.conf 文件  
在\[lvm\]部分，配置LVM后端以LVM驱动结束，卷组cinder-volumes，iSCSI协议和正确的iSCSI服务  
\[lvm\]  
volume\_driver = cinder.volume.drivers.lvm.LVMVolumeDriver \# 驱动  
volume\_group = cinder-volumes \# vg组名称  
iscsi\_protocol = iscsi \# iSCSI协议  
iscsi\_helper = lioadm \# iSCSI管理工具  
volume\_backend\_name=iSCSI-Storage \# 名称在 \[DEFAULT\] 区域，配置镜像服务 API 的位置  
在\[DEFAULT\]部分，启用 LVM 后端  
\[DEFAULT\]  
enabled\_backends = lvm  
在\[DEFAULT\]区域，配置镜像服务 API 的位置  
\[DEFAULT\]  
glance\_api\_servers = http://192.168.137.11:9292

8\)启动块存储卷服务及其依赖的服务，并将其配置为随系统启动  
systemctl enable openstack-cinder-volume.service target.service  
systemctl restart openstack-cinder-volume.service target.service

9\)使用cinder create 命令创建卷，再通过nova volume-attach 命令把卷改在到虚机上  
10\)在存储节点上执行lvdisplay命令，查看刚才挂载的数据卷

