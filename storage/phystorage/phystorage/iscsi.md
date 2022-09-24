## iSCSI

原来只用于本机的SCSI协义透过TCP/IP网络发送就是iSCSI协议。

iSCSI target是位于互联网上服务器上的存储资源。

**被访问的设备称为Target，而访问Target 称为Initiator。**![](/assets/storage-hardware-proto-iscsi1.png)说白了，就是把存储资源接入网络，在网络上用SCSI协议传输数据，通过一定的机制，使得Target接入网络后能被Initiator发现，Initiator通过网络直接连Target读写数据。

## ISER和iSCSI Target的关系

iSCSI target是位于互联网上服务器上的存储资源。

原来 客户端和target 传输数据的时候走的是IP/TCP协议栈，就是iSCSI。

现在换成走RDMA协议栈，可以在用RDMA 网络传输的SCSI，就是ISER。



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



