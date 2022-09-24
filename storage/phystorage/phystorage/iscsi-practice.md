# iSCSI实战

## **所需软件与软件结构**

CentOS 将 tgt 的软件名称定义为 scsi-target-utils ，因此你得要使用 yum 去安装他才行。至于用来作为 initiator 的软件则是使用 linux-iscsi 的项目，该项目所提供的软件名称则为 iscsi-initiator-utils 。所以，总的来说，你需要的软件有：

* scsi-target-utils：用来将 Linux 系统仿真成为 iSCSI target 的功能；
* iscsi-initiator-utils：挂载来自 target 的磁盘到 Linux 本机上。

那么 scsi-target-utils 主要提供哪些档案呢？基本上有底下几个比较重要需要注意的：

* /etc/tgt/targets.conf：主要配置文件，设定要分享的磁盘格式与哪几颗；
* /usr/sbin/tgt-admin：在线查询、删除 target 等功能的设定工具；
* /usr/sbin/tgt-setup-lun：建立 target 以及设定分享的磁盘与可使用的

_客户端等工具软件。_

* /usr/sbin/tgtadm：手动直接管理的管理员工具 \(可使用配置文件取代\)；
* /usr/sbin/tgtd：主要提供 iSCSI target 服务的主程序；
* /usr/sbin/tgtimg：建置预计分享的映像文件装置的工具 \(以映像文件仿真磁盘\)；

其实 CentOS 已经将很多功能都设定好了，因此我们只要修订配置文件，然后启动 tgtd 这个服务就可以啰！ 接下来，就让我们实际来玩一玩 iSCSI target 的设定吧！

## **target 的实际设定**

从上面的分析来看，iSCSI 就是透过一个网络接口，将既有的磁盘给分享出去就是了。那么有哪些类型的磁盘可以分享呢？ 这包括：

* 使用 dd 指令所建立的大型档案可供仿真为磁盘 \(无须预先格式化\)；
* 使用单一分割槽 \(partition\) 分享为磁盘；
* 使用单一完整的磁盘 \(无须预先分割\)；
* 使用磁盘阵列分享 \(其实与单一磁盘相同方式\)；
* 使用软件磁盘阵列 \(software raid\) 分享成单一磁盘；
* 使用 LVM 的 LV 装置分享为磁盘。

## **建立所需要的磁盘装置**

既然 iSCSI 要分享的是磁盘，那么我们得要准备好啊！目前预计准备的磁盘为：

* 建立一个名为 /srv/iscsi/disk1.img 的 500MB 档案；
* 使用 /dev/sda10 提供 2GB 作为分享 \(从第一章到目前为止的分割数\)；
* 使用 /dev/server/iscsi01 的 2GB LV 作为分享 \(再加入 5GB /dev/sda11 到 server VG 中\)。

实际处理的方式如下：

```
# 1. 建立大型档案：
[root@www ~]# mkdir /srv/iscsi 
[root@www ~]# dd if=/dev/zero of=/srv/iscsi/disk1.img bs=1M count=500 
[root@www ~]# chcon -Rv -t tgtd_var_lib_t /srv/iscsi/ 
[root@www ~]# ls -lh /srv/iscsi/disk1.img -rw-r--r--. 1 root root 500M Aug 2 16:22 /srv/iscsi/disk1.img <==容量对的！ # 2. 建立实际的 partition 分割： 
[root@www ~]# fdisk /dev/sda <==实际的分割方式自己处理吧！ [root@www ~]# partprobe <==某些情况下得 reboot 喔！ [root@www ~]# fdisk -l Device Boot Start End Blocks Id System /dev/sda10 2202 2463 2104483+ 83 Linux /dev/sda11 2464 3117 5253223+ 8e Linux LVM # 只有输出 /dev/sda{10,11} 信息，其他的都省略了。注意看容量，上述容量单位 KB 
[root@www ~]# swapon -s; mount | grep 'sda1' # 自己测试一下 /dev/sda{10,11} 不能够被使用喔！若有被使用，请 umount 或 swapoff # 3. 建立 LV 装置 ： 
[root@www ~]# pvcreate /dev/sda11 [root@www ~]# vgextend server /dev/sda11 
[root@www ~]# lvcreate -L 2G -n iscsi01 server 
[root@www ~]# lvscan ACTIVE '/dev/server/myhome' [6.88 GiB] inherit ACTIVE '/dev/server/iscsi01' [2.00 GB] inherit
```

## **规划分享的 iSCSI target 檔名**

iSCSI 有一套自己分享 target 档名的定义，基本上，藉由 iSCSI 分享出来的 target 檔名都是以 iqn 为开头，意思是：『iSCSI Qualified Name \(iSCSI 合格名称\)』的意思。那么在 iqn 后面要接啥档名呢？通常是这样的：

```
iqn.yyyy-mm.<reversed domain name>:identifier 
iqn.年年-月.单位网域名的反转写法 :这个分享的target名称
```

我做这个测试的时间是 2014 年 10 月份，然后鸟哥的机器是 www.vlnb.net ，反转网域写法为 net.vlnb， 然后，想要的 iSCSI target 名称是 vdisk ，那么就可以这样写：

```
iqn.2014-010.net.vlnb:vdisk
```

另外，就如同一般外接式储存装置 \(target 名称\) 可以具有多个磁盘一样，我们的 target 也能够拥有数个磁盘装置的。 每个在同一个 target 上头的磁盘我们可以将它定义为逻辑单位编号 \(Logical Unit Number, LUN\)。我们的 iSCSI initiator 就是跟 target 协调后才取得 LUN 的存取权就是了。在这个简单案例中，最终的结果，我们会有一个 target ，在这个 target 当中可以使用三个 LUN 的磁盘。

**设定 tgt 的配置文件 /etc/tgt/targets.conf**

接下来我们要开始来修改配置文件了。基本上，配置文件就是修改 /etc/tgt/targets.conf 啦。这个档案的内容可以改得很简单， 最重要的就是设定前一点规定的 iqn 名称，以及该名称所对应的装置，然后再给予一些可能会用到的参数而已。 多说无益，让我们实际来实作看看：

```
[root@www ~]# vim /etc/tgt/targets.conf 
# 此档案的语法如下： 
<target iqn.相关装置的target名称>
 　　backing-store /你的/虚拟设备/完整檔名-1
 　　backing-store /你的/虚拟设备/完整檔名-2 
</target> 
<target iqn.2014-10.net.vlnb:vdisk>
 　　backing-store /srv/iscsi/disk1.img <==LUN 1 (LUN 的编号通常照顺序)         　　 backing-store /dev/sda10 <==LUN 2
　　 backing-store /dev/server/iscsi01 <==LUN 3 
</target>
```

事实上，除了 backing-store 之外，在这个配置文件当中还有一些比较特别的参数可以讨论看看 \(man tgt-admin\)：

* backing-store \(虚拟的装置\), direct-store \(实际的装置\)： 设定装置时，如果你的整颗磁盘是全部被拿来当 iSCSI 分享之用，那么才能够使用 direct-store 。不过，根据网络上的其他文件， 似乎说明这个设定值有点危险的样子。所以，基本上还是建议单纯使用模拟的 backing-store 较佳。例如鸟哥的简单案例中，就通通使用 backing-store 而已
* initiator-address \(用户端地址\)： 如果你想要限制能够使用这个 target 的客户端来源，才需要填写这个设定值。基本上，不用设定它 \(代表所有人都能使用的意思\)， 因为我们后来会使用 iptables 来规范可以联机的客户端嘛！
* incominguser \(用户账号密码设定\)： 如果除了来源 IP 的限制之外，你还想要让使用者输入账密才能使用你的 iSCSI target 的话，那么就加用这个设定项目。 此设定后面接两个参数，分别是账号与密码啰。
* write-cache \[off\|on\] \(是否使用快取\)： 在预设的情况下，tgtd 会使用快取来增快速度。不过，这样可能会有遗失数据的风险。所以，如果你的数据比较重要的话， 或许不要使用快取，直接存取装置会比较妥当一些。

上面的设定值要怎么用呢？现在，假设你的环境中，仅允许 192.168.100.0/24 这个网段可以存取 iSCSI target，而且存取时需要帐密分别为 vuser, vpasswd ，此外，不要使用快取，那么原本的配置文件之外，还得要加上这样的参数才行 \(基本上，使用上述的设定即可，底下的设定是多加测试用的，不需要填入你的设定中\)。

```
[root@www ~]# vim /etc/tgt/targets.conf 
<target iqn.2014-10.net.vlnb:vdisk> 
　　backing-store /home/iscsi/disk1.img 
　　backing-store /dev/sda7 
　　backing-store /dev/server/iscsi01
 　 initiator-address 192.168.100.0/24
   incominguser vbirduser vbirdpasswd 
   write-cache off 
</target>
```

**启动 iSCSI target 以及观察相关端口口与磁盘信息**  
再来则是启动、开机启动，以及观察 iSCSI target 所启动的埠口啰：

```
[root@www ~]# /etc/init.d/tgtd start 
[root@www ~]# chkconfig tgtd on 
[root@www ~]# netstat -tlunp | grep tgt 
Active Internet connections (only servers) 
Proto Recv-Q Send-Q Local Address Foreign Address State PID/Program name 
tcp 0 0 0.0.0.0:3260 0.0.0.0:* LISTEN 
26944/tgtd tcp 0 0 :::3260 :::* LISTEN 
26944/tgtd 
# 重点就是那个 3260 TCP 封包啦！等一下的防火墙务必要开放这个埠口。 
# 观察一下我们 target 相关信息，以及提供的 LUN 数据内容： [root@www ~]# tgt-admin --show 
Target 1: iqn.2011-08.vbird.centos:vbirddisk <==就是我们的 target 
　　System information: 
　　　　Driver: iscsi 
　　　　State: ready 
　　I_T nexus information:
　　LUN information: 
　　　　LUN: 0 
　　　　　　Type: controller <==这是个控制器，并非可以用的 LUN 喔！
 　　　　....(中间省略)....
　　LUN: 1 
　　　　Type: disk <==第一个 LUN，是磁盘 (disk) 喔！ 
　　　　SCSI ID: IET 00010001 
　　　　SCSI SN: beaf11 
　　　　Size: 2155 MB <==容量有这么大！ 
　　　　Online: Yes 
　　　　Removable media: No Backing store 
　　　　type: rdwr 
　　　　Backing store path: /dev/sda10 <==磁盘所在的实际文件名 
　　LUN: 2 
　　　　Type: disk 
　　　　SCSI ID: IET 00010002 
　　　　SCSI SN: beaf12 
　　　　Size: 2147 MB 
　　　　Online: Yes 
　　　　Removable media: No 
　　　　Backing store type: rdwr 
　　　　Backing store path: /dev/server/iscsi01 
　　　LUN: 3
　　　　Type: disk 
　　　　SCSI ID: IET 00010003 
　　　　SCSI SN: beaf13 
　　　　Size: 524 MB 
　　　　Online: Yes 
　　　　Removable media: No 
　　　　Backing store type: rdwr 
　　　　Backing store path: /srv/iscsi/disk1.img 
Account information: 
　　vuser <==额外的帐户信息 
ACL information: 
　　192.168.100.0/24 <==额外的来源 IP 限制
```

请将上面的信息对照一下我们的配置文件呦！看看有没有错误就是了！尤其注意每个 LUN 的容量、实际磁盘路径！ 那个项目不能错误就是了。\(照理说 LUN 的数字应该与 backing-store 设定的顺序有关，不过，在鸟哥的测试中， 出现的顺序并不相同！因此，还是需要使用 tgt-admin --show 去查阅查阅才好！\)

**设定防火墙**

不论你有没有使用 initiator-address 在 targets.conf 配置文件中，iSCSI target 就是使用 TCP/IP 传输数据的， 所以你还是得要在防火墙内设定可以联机的客户端才行！既然 iSCSI 仅开启 3260 埠口，那么我们就这么进行即可：可以参考[http://www.111cn.net/sys/linux/44803.htm](http://www.111cn.net/sys/linux/44803.htm)

```
[root@www ~]# vi /etc/sysconfig/iptables
iptables -A INPUT -p tcp -s 192.168.100.0/24 --dport 3260 -j ACCEPT
```

**iSCSI initiator 的设定**

谈完了 target 的设定，并且观察到相关 target 的 LUN 数据后，接下来就是要来挂载使用啰。使用的方法很简单， 只不过我们得要安装额外的软件来取得 target 的 LUN 使用权就是了。

**所需软件与软件结构**

在前一小节就谈过了，要设定 iSCSI initiator 必须要安装 iscsi-initiator-utils 才行。安装的方法请使用 yum 去处理，这里不再多讲话。那么这个软件的结构是如何呢？

* /etc/iscsi/iscsid.conf：主要的配置文件，用来连结到 iSCSI target 的设定；
* /sbin/iscsid：启动 iSCSI initiator 的主要服务程序；
* /sbin/iscsiadm：用来管理 iSCSI initiator 的主要设定程序；
* /etc/init.d/iscsid：让本机模拟成为 iSCSI initiater 的主要服务；
* /etc/init.d/iscsi：在本机成为 iSCSI initiator 之后，启动此脚本，让我们可以登入 iSCSI target。所以 iscsid 先启动后，才能启动这个服务。为了防呆，所以 /etc/init.d/iscsi 已经写了一个启动指令， 启动 iscsi 前尚未启动 iscsid ，则会先呼叫 iscsid 才继续处理 iscsi 喔
  _！_

老实说，因为 /etc/init.d/iscsi 脚本已经包含了启动 /etc/init.d/iscsid 的步骤在里面，所以，理论上， 你只要启动 iscsi 就好啦！此外，那个 iscsid.conf 里面大概只要设定好登入 target 时的帐密即可， 其他的 target 搜寻、设定、取得的方法都直接使用 iscsiadm 这个指令来完成。由于 iscsiadm 侦测到的结果会直接写入 /var/lib/iscsi/nodes/ 当中，因此只要启动 /etc/init.d/iscsi 就能够在下次开机时，自动的连结到正确的 target 啰。 那么就让我们来处理处理整个过程吧

**initiator 的实际设定**

首先，我们得要知道 target 提供了啥咚咚啊，因此，理论上，不论是 target 还是 initiator 都应该是要我们管理的机器才对。 而现在我们知道 target 其实有设定账号与密码的，所以底下我们就得要修改一下 iscsid.conf 的内容才行。

修改 /etc/iscsi/iscsid.conf 内容，并启动 iscsi

这个档案的修改很简单，因为里面的参数大多已经预设做的不错了，所以只要填写 target 登入时所需要的帐密即可。 修改的地方有两个，一个是侦测时 \(discovery\) 可能会用到的帐密，一个是联机时 \(node\) 会用到的帐密：

```
[root@clientlinux ~]# vim /etc/iscsi/iscsid.conf
node.session.auth.username = vbirduser <==在 target 时设定的 
node.session.auth.password = vbirdpasswd <==约在 53, 54 行 
discovery.sendtargets.auth.username = vbirduser <==约在 67, 68 行 
discovery.sendtargets.auth.password = vbirdpasswd 
[root@clientlinux ~]# chkconfig iscsid on 
[root@clientlinux ~]# chkconfig iscsi on
```

由于我们尚未与 target 联机，所以 iscsi 并无法让我们顺利启动的！因此上面只要 chkconfig 即可，不需要启动他。 要开始来侦测 target 与写入系统信息啰。全部使用 iscsiadm 这个指令就可以完成所有动作了。

侦测 192.168.100.254 这部 target 的相关数据  
虽然我们已经知道 target 的名字，不过，这里假设还不知道啦！因为有可能哪一天你的公司有钱了， 会去买实体的 iSCSI 数组嘛！所以这里还是讲完整的侦测过程好了！你可以这样使用：

```
[root@clientlinux ~]# iscsiadm -m discovery -t sendtargets -p IP:port
选项与参数： 
-m discovery ：使用侦测的方式进行 iscsiadmin 指令功能； 
-t sendtargets ：透过 iscsi 的协议，侦测后面的设备所拥有的 target 数据 -p IP:port ：就是那部 iscsi 设备的 IP 与埠口，不写埠口预设是 3260 啰！

范例：侦测 192.168.100.254 这部 iSCSI 设备的相关数据 [root@clientlinux ~]# iscsiadm -m discovery -t sendtargets -p 192.168.100.254
192.168.100.254:3260,1 iqn.2014-10.net.vbnl:vdisk
# 192.168.100.254:3260,1 ：在此 IP, 端口口上面的 target 号码，本例中为 target1
#iqn.2014-10.net.vbnl:vdisk ：就是我们的 target 名称啊！

[root@clientlinux ~]# ll -R /var/lib/iscsi/nodes/
/var/lib/iscsi/nodes/iqn.2014-10.net.vbnl:vdisk
/var/lib/iscsi/nodes/iqn.2014-10.net.vbnl:vdisk/192.168.100.254,3260,1
# 上面的特殊字体部分，就是我们利用 iscsiadm 侦测到的 target 结果！
```

现在我们知道了 target 的名称，同时将所有侦测到的信息通通写入到上述 /var/lib/iscsi/nodes/iqn.2014-10.net.vbnl:vbirddisk/192.168.100.254,3260,1 目录内的 default 档案中， 若信息有修订过的话，那你可以到这个档案内修改，也可以透过 iscsiadm 的 update 功能处理相关参数的。

**开始进行联机 iSCSI target**

因为我们的 initiator 可能会连接多部的 target 设备，因此，我们得先要瞧瞧目前系统上面侦测到的 target 有几部， 然后再找到我们要的那部 target 来进行登入的作业。不过，如果你想要将所有侦测到的 target 全部都登入的话， 那么整个步骤可以再简化：

```
范例：根据前一个步骤侦测到的资料，启动全部的 target 
[root@clientlinux ~]# /etc/init.d/iscsi restart 
正在停止 iscsi： [ 确定 ] 
正在激活 iscsi： [ 确定 ] 
# 将系统里面全部的 target 通通以 /var/lib/iscs/nodes/ 内的设定登入 
# 上面的特殊字体比较需要注意啦！你只要做到这里即可，底下的瞧瞧就好。 范例：显示出目前系统上面所有的 target 数据： 
[root@clientlinux ~]# iscsiadm -m node
192.168.100.254:3260,1 iqn.2014-10.net.vbnl:vdisk
选项与参数： 
-m node：找出目前本机上面所有侦测到的 target 信息，可能并未登入喔 

范例：仅登入某部 target ，不要重新启动 iscsi 服务 
[root@clientlinux ~]# iscsiadm -m node -T target名称 --login 
选项与参数： 
-T target名称：仅使用后面接的那部 target ，target 名称可用上个指令查到！ 
--login ：就是登入啊！

[root@clientlinux ~]# iscsiadm -m node -T iqn.iqn.2014-10.net.vbnl:vdisk \ 
> --login 
# 这次进行会出现错误，是因为我们已经登入了，不可重复登入喔！
```

接下来呢？呵呵！很棒的是，我们要来开始处理这个 iSCSI 的磁盘了喔！怎么处理？瞧一瞧！

```
[root@clientlinux ~]# fdisk -l 
Disk /dev/sda: 8589 MB, 8589934592 bytes <==这是原有的那颗磁盘，略过不看
 ....(中间省略).... 
Disk /dev/sdc: 2147 MB, 2147483648 bytes 
67 heads, 62 sectors/track, 1009 cylinders 
Units = cylinders of 4154 * 512 = 2126848 bytes 
Sector size (logical/physical): 512 bytes / 512 bytes 

Disk /dev/sdb: 2154 MB, 2154991104 bytes 
67 heads, 62 sectors/track, 1013 cylinders 
Units = cylinders of 4154 * 512 = 2126848 bytes 
Sector size (logical/physical): 512 bytes / 512 bytes 

Disk /dev/sdd: 524 MB, 524288000 bytes 
17 heads, 59 sectors/track, 1020 cylinders 
Units = cylinders of 1003 * 512 = 513536 bytes 
Sector size (logical/physical): 512 bytes / 512 bytes
```

你会发现主机上面多出了三个新的磁盘，容量与刚刚在 192.168.100.254 那部 iSCSI target 上面分享的 LUN 一样大。 那这三颗磁盘可以怎么用？你想怎么用就怎么用啊！只是，唯一要注意的，就是 iSCSI target 每次都要比 iSCSI initiator 这部主机还要早开机，否则我们的 initiator 恐怕就会出问题。

**更新/删除/新增 target 数据的方法**

```
[root@clientlinux ~]# iscsiadm -m node -T targetname --logout [root@clientlinux ~]# iscsiadm -m node -o [delete|new|update] -T targetname 
选项与参数： --logout ：就是注销 target，但是并没有删除 /var/lib/iscsi/nodes/ 内的数据
-o delete：删除后面接的那部 target 链接信息 (/var/lib/iscsi/nodes/*) 
-o update：更新相关的信息
-o new ：增加一个新的 target 信息。 

范例：关闭来自鸟哥的 iSCSI target 的数据，并且移除链接 
[root@clientlinux ~]# iscsiadm -m node <==还是先秀出相关的 target iqn 名称 
192.168.100.254:3260,1 iqn.2014-10.net.vbnl:vdisk 
[root@clientlinux ~]# iscsiadm -m node -T iqn.2014-10.net.vbnl:vdisk \ 
> --logout 
Logging out of session [sid: 1, target: 
iqn.2011-08.vbird.centos:vbirddisk, portal: 192.168.100.254,3260] 
Logout of [sid: 1, target: iqn.2014-10.net.vbnl:vdisk, portal: 192.168.100.254,3260] successful. 
# 这个时候的 target 连结还是存在的，虽然注销你还是看的到！ 

[root@clientlinux ~]# iscsiadm -m node -o delete \ 
> -T iqn.iqn.2014-10.net.vbnl:vdisk
[root@clientlinux ~]# iscsiadm -m node 
iscsiadm: no records found! <==嘿嘿！不存在这个 target 了～ 
[root@clientlinux ~]# /etc/init.d/iscsi restart 
# 你会发现唔！怎么 target 的信息不见了！这样瞭了乎
```

如果一切都没有问题，现在，请回到 discovery 的过程，重新再将 iSCSI target 侦测一次，再重新启动 initiator 来取得那三个磁盘吧！我们要来测试与利用该磁盘啰！

