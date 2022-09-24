## iSCSI

原来只用于本机的SCSI协义透过TCP/IP网络发送就是iSCSI协议。

iSCSI target是位于互联网上服务器上的存储资源。

**被访问的设备称为Target，而访问Target 称为Initiator。**![](/assets/storage-hardware-proto-iscsi1.png)说白了，就是把存储资源接入网络，在网络上用SCSI协议传输数据，通过一定的机制，使得Target接入网络后能被Initiator发现，Initiator通过网络直接连Target读写数据。

![](/assets/storage-hardware-iscsi2.png)

## ISER和iSCSI Target的关系

iSCSI target是位于互联网上服务器上的存储资源。

原来 客户端和target 传输数据的时候走的是IP/TCP协议栈，就是iSCSI。

现在换成走RDMA协议栈，可以在用RDMA 网络传输的SCSI，就是ISER。

## 几种常见的 iSCSI Target

**STGT**

Linux SCSI target framework \(tgt\) aims to simplify various SCSI target driver \(iSCSI, Fibre Channel, SRP, etc\) creation and maintenance. Our key goals are the clean integration into the scsi-mid layer and implementing a great portion of tgt in user space.

MainPage: Linux SCSI target framework \(tgt\) project

GitHub: https://github.com/fujita/tgt

Quickstart: Scsi-target-utils Quickstart Guide - Fedora Project Wiki

**SCST**

The generic SCSI target subsystem for Linux \(SCST\) allows creation of sophisticated storage devices from any Linux box. Those devices can provide advanced functionality, like replication, thin provisioning, deduplication, high availability, automatic backup, etc.

MainPage: SCST: A Generic SCSI Target Subsystem for Linux

**LIO**

LinuxIO \(LIO™\) is the standard open-source SCSI target in Linux. It supports all prevalent storage fabrics, including Fibre Channel \(QLogic, Emulex\), FCoE, iEEE 1394, iSCSI \(incl. Chelsio offload support\), NVMe-OF, iSER \(Mellanox InfiniBand\), SRP \(Mellanox InfiniBand\), USB, vHost, etc.

MainPage: Linux SCSI Target - Main Page

### 优缺点比较

（1）STGT

tgt 是一个用户态的 SCSI target 框架，在 GNU/Linux 内核直接集成 SCSI target 框架之前，这是一个绝对主流的框架。

优点：

1）简单，方便使用和维护。

2）另外已经有 ceph 的 target driver，只是需要做性能优化。

3）因为工作在用户态，所以即使挂掉了，也不会对其他运行的程序产生影响。



缺点：

1）支持的传输协议较少。

2）对 SCSI 协议支持比较简单，一些 cluster 中的特性比如 PR 等都不支持，所以基于 stgt 的方案不能在 cluster 中使。

3）由于是用户态框架，性能问题较差，根据网上的相关数据， tgt 在使用本地存储的情况下，性能相比后面会提到的 SCST、 LIO 等是有一定差距的。



（2）SCST

SCST 的核心模块工作在内核里，可以支持通过系统模块（VFS、块层）访问的后端存储如块设备、文件设备以及 passthrough 的 scsi 设备。

优点：

1）支持更多传输协议。

2）针对性能做了特殊的优化。

3）除了基本的 SCSI 协议支持外，还有一些高级支持：

SCST支持永久性预留（Persistent Reservation, PR）；这是一个用于高可用集群中的存储设备的 I/O 隔离与存储设备故障切换、接管的特性。通过使用 PR 命令，initiator 可以在一个 target 上建立、抢占、查询、重置预留策略。在故障接管过程中，新的虚拟资源可以重置老的虚拟资源的预留策略，从而让故障切换更快、更容易地进行。

SCST 可以使用异步事件通知（AEN）来通告会话状态的变更。AEN 是一个 SCSI target 用来向 initiator 进行 target 端的事件告知的协议特性，即使在没有服务请求的时候也可以进行。于是 initiator 就可以在 target 端发生事件时，如设备插入、移除、调整尺寸或更换介质时，可以得到通知。这让 initiator 可以以即插即用的方式看到 target 的变化。

4）SCST 的开发者声称，它们的设计在健壮性和安全性方面更加符合 SCSI 标准。SCSI 协议要求，如果一个 initiator 要清除另一个 initiator 的预留资源时，预留者必须要得到清除通知，否则，多个 initiator 都可能来改变预留数据，就可能会破坏数据。SCST 可以实现安全的预留、释放操作，避免类似事情发生。

5）SCST 也支持非对称逻辑卷分配（ALUA）。ALUA 允许 target 管理员来管理 target 的访问状态和路径属性。这让多路径路由机制可以选择最好的路径，从而根据 target 的访问状态，优化带宽的使用。换句话说，在多路径环境下，target 管理员可以通过改变访问状态来调整 initiator 的路径。

6）各大存储服务提供商都是基于 SCST。

7）提供更细粒度的访问控制策略以及 QoS 保证机制（限制 initiator 连接的个数）。



缺点：

1）结构复杂，二次开发成本较高。

2）工作在 kernel，如果挂了，会导致整个机器 down 掉，影响其他程序。

3）kernel 部分没有并入 linux，需要手工编译。



（3）LIO

LIO 也即 Linux-IO，是目前 GNU/Linux 内核自带的 SCSI target 框架（自 2.6.38版本开始引入，真正支持 iSCSI 需要到 3.1 版本） ，对 iSCSI RFC 规范的支持非常好，包括完整的错误恢复都有支持。整个 LIO 是纯内核态实现的，包括前端接入和后端存储模块，为了支持用户态后端，从内核 3.17 开始引入用户态后端支持，即 TCMU\(Target Core Module in Userspace\)。



优点：

1）支持较多传输协议。

2）代码并入 linux 内核，减少了手动编译内核的麻烦。

3）提供了python版本的编程接口 rtslib。

4）LIO 在不断 backport SCST 的功能到 linux 内核，社区的力量是强大的。

5）LIO 也支持一些 SCST 没有的功能。如 LIO 还支持“会话多连接”（MC/S）。

6）LIO 支持最高级别的 ERL。



缺点：

1）不支持 AEN，所以 target 状态发生变化时，只能通过 IO或者用户手动触发以检测处理变化。

2）结构相对复杂，二次开发成本较高。

3）工作在内核态，出现问题会影响其他程序的运行。



**软iscsi target**

检查系统是否安装 scsi-target  

//用来将Linux 系统模拟成为iSCSI target 的功能,也就是常说的软iscsi target. 



## iSCSI实战

[详细实战案例](storage/phystorage/phystorage/iscsi-practice.md)

1. 安装软件包

yum -y install iscsi-initiator-utils

2.1 发现启动器命令

iscsiadm -m discovery -t sendtargets -p 192.168.1.3

结果：192.168.255.30:3260,1 iqn.20080-03.com.30:storage.iscsitest   \#红色字体为iscsi initiator名称，登记过程中会用到

注：-p 后面IP为SVIP或VIP，如果是SVIP就返回所有先关映射的VIP target信息，如果是VIP就返回对应一个vip对应Target信息

2.2 连接（login）

iscsiadm -m node -T iqn.20080-03.com.30:storage.iscsitest -l

反馈结果：Login session \[iface: default, target: iqn.20080-03.com.30:storage.iscsitest, portal: 192.168.255.30,3260\]

注：-T 是扫描到的target信息

2.3 开机自动连接配置\(测试没必要配置\)

修改/etc/iscsi/iscsid.conf文件，将：\#node.startup = automatic 一行前面的\#去掉改成node.startup = automatic

```
chkconfig iscsi on

chkconfig iscsid on
```

2.4 断开（logout）

```
  iscsiadm -m node -T iqn.20080-03.com.30:storage.iscsitest -p 192.168.255.30 -u
```

断开指定Target：

iscsiadm -m node -T iqn.20080-03.com.30:storage.iscsitest --logout

2.5 查看iscsi发现记录

iscsiadm -m node

2.6 删除iscsi发现记录

iscsiadm -m node -o delete -T iqn.20080-03.com.30:storage.iscsitest -p 192.168.255.30

2.7 连接所有target

iscsiadm -m node -L all

2.8 查看目前 iSCSI target  连接 状态

iscsiadm -m session

2.9 断开所有 Target 连接

iscsiadm -m node -u

2.10 刪除所有 node  信息  \( 需重新  discovery\)

iscsiadm -m node -o delete

注：断开连接最好把2.9和2.10步骤都执行一下，清空所有信息

2.11查看数据结构的树状信息

```
  iscsiadm -m node -o show -T iqn.20080-03.com.30:storage.iscsitest
```



