## 前言 {#前言}

在使用云服务器产品时，由于问题修复、功能添加、软件更新等原因，往往需要对已有系统镜像进行二次修改。

这种情况下，最为简单的做法是：使用原镜像创建实例，在实例中进行修改，修改完毕后再转镜像。这种做法比较粗暴，系统启动的过程可能会为原本干净的系统镜像带来一些不必要的残留数据，而且也无法执行硬盘分区、文件系统等需要目标文件系统不在挂载状态下才能够执行的操作。

针对这些问题，一种改善的方法是：使用原镜像创建一块云硬盘，随后将这块云硬盘挂载至一台已创建的实例上并进行修改，修改完毕后将云硬盘转镜像。这样能够在较大程度上保证系统镜像的纯净性，也能够进行分区、文件系统方面的修改。

不过上述方式整体操作流程仍然比较冗长，为了缩减流程，libvirt提供了名为libguestfs-tools的工具，用来快捷地操作硬盘镜像。

## 简介 {#简介}

libguestfs-tools是一款系统管理员用来管理虚拟机硬盘镜像的工具包，里面包含了用于各种各样用途的工具，能够帮忙我们操作硬盘镜像的分区、文件系统、文件等等。

libguestfs-tools包含了如下工具：

1. guestfish提供了一个交互式shell，在这个shell中可以通过一系列内置命令操作硬盘镜像的分区、文件系统、文件等等。
2. guestmount能够将硬盘镜像中的文件系统挂载至宿主机上，从而像操作宿主机的文件一样操作硬盘镜像中的文件。
3. virt-alignment-scan能够扫描磁盘镜像并检测分区对齐问题。
4. virt-builder用于快速地构建一个可用的硬盘镜像。
5. virt-cat用来展示硬盘镜像中指定文件的内容。
6. virt-copy-in和virt-copy-out用来上传/下载硬盘镜像中的文件/目录。
7. virt-customize用来定制硬盘镜像。
8. virt-df展示硬盘镜像空间的使用情况。
9. virt-diff展示硬盘镜像内两个文件的差异。
10. virt-edit用来编辑硬盘镜像中的文件。
11. virt-filesystems展示硬盘镜像中的文件系统、硬盘分区、块设备、LVM信息等等。
12. virt-format用来格式化硬盘镜像内部的文件系统。
13. virt-get-kernel用来获取硬盘镜像中的kernel/initrd。
14. virt-inspector用来探测硬盘镜像内部的系统信息。
15. virt-log展示硬盘镜像中的日志。
16. virt-ls用来列出硬盘镜像中的文件。
17. virt-make-fs能够使用指定的文件或归档包创建包含有一个文件系统的硬盘镜像。
18. virt-rescue提供了一个交互式shell，用于操作硬盘镜像。
19. virt-resize能够扩容硬盘镜像中的分区。
20. virt-sparsify能够消除硬盘镜像中的空洞，压缩镜像大小。
21. virt-syspgrep用于镜像初始化。
22. virt-tail能够追踪硬盘镜像中指定文件的更新情况。
23. virt-tar-in和virt-tar-out能够上传/下载硬盘镜像中的文件/目录并归档。
24. virt-win-reg能够修改windows硬盘镜像中的注册表。

如上，可以看到为了简化对硬盘镜像的操作，libguestfs-tools提供了非常非常多的工具，这些工具每个都有着各自的用途，有的十分好用，而有的却很鸡肋。在绝大多数情况下，**guestmount**工具就可以满足我们的需求。

## 安装 {#安装}

libguestfs-tools工具位于centos的base源中，确认软件源后安装libguestfs-tools工具包：

shell

```
yum install -y libguestfs-tools

```

## 案例 {#案例}

### 文件系统、分区、块设备、LVM信息获取 {#文件系统分区块设备lvm信息获取}

通过**virt-filesystems**命令可以方便地获取到硬盘镜像中所有分区和文件系统的信息：

shell

```
export LIBGUESTFS_BACKEND=direct

virt-filesystems --all --long --uuid -h -a centos.qcow2

Name                         Type       VFS  Label MBR Size Parent                  UUID

/dev/sda1                    filesystem xfs  -     -   1.0G -                       c21cde6e-fe3a-4aa1-8adb-d26d096b1889

/dev/sda2                    filesystem swap -     -   2.0G -                       ef8945c2-3538-4adf-a70c-c46d339b150d

/dev/centos_hikvisionos/root filesystem xfs  -     -   17G  -                       70ced207-d3bc-4ca0-b3ea-3050ebd4990c

/dev/centos_hikvisionos/root lv         -    -     -   17G  /dev/centos_hikvisionos gjQuy8-ed48-WMDo-PROV-14Sl-ghdp-UGE0b9

/dev/centos_hikvisionos      vg         -    -     -   17G  /dev/sda3               BXeW80Nd2VQNBnvOCypsbf0a0n8etjSQ

/dev/sda3                    pv         -    -     -   17G  -                       ZuUKoHbV7jyJW6C57p36BXvOuUzz4BRB

/dev/sda1                    partition  -    -     83  1.0G /dev/sda                -

/dev/sda2                    partition  -    -     82  2.0G /dev/sda                -

/dev/sda3                    partition  -    -     83  17G  /dev/sda                -

/dev/sda                     device     -    -     -   20G  -                       -

```

### \* 文件修改 {#-文件修改}

我们对硬盘镜像的改动在绝大多数情况下都属于文件修改，而不需要涉及到分区或文件系统的情况下，使用**guestmount**命令可以轻松搞定。

shell

```
#
 镜像的挂载路径
MOUNT_PATH=/mnt/centos

#
 镜像的路径
IMAGE=/root/centos.qcow2

export LIBGUESTFS_BACKEND=direct

#
 -a指定镜像，-i表示自动识别待挂载的镜像文件系统
guestmount -a $IMAGE -i $MOUNT_PATH

#
 -m手动指定镜像内的待挂载文件系统
#
 guestmount -a 
$IMAGE
 -m /dev/centos/root 
$MOUNT_PATH
#
 进入挂载路径
cd $MOUNT_PATH

```

上面这种方式能够修改硬盘镜像中的文件，不过，却无法执行硬盘镜像中的命令，如果涉及到需要执行镜像中命令的操作，比如说yum install安装软件、grub-install生成grub配置、dracut生成initrd等等，还需要增加一步chroot。

shell

```
#
 挂载必要的设备路径并chroot
mount -t proc none $MOUNT_PATH/proc

mount --bind /dev $MOUNT_PATH/dev

mount -t devpts devpts $MOUNT_PATH/dev/pts

mount -t sysfs none $MOUNT_PATH/sys

chroot $MOUNT_PATH

#
 执行操作
rm -rf /tmp/*

yum clean all

rm -rf /var/cache/yum

cd /var/log/; ls -I audit -I sa -I samba|xargs rm -rf

```

在操作完后，卸载镜像挂载路径，对镜像的改动就算作是完成了，如果之前有chroot，那么还需退出chroot并卸载相关设备路径。

shell

```
#
 退出chroot并卸载设备路径
exit

rm -f $MOUNT_PATH/root/.bash_history

umount $MOUNT_PATH/proc

umount $MOUNT_PATH/dev/pts

umount $MOUNT_PATH/dev

umount $MOUNT_PATH/sys

#
 卸载镜像挂载路径
umount $MOUNT_PATH

```

### 分区移除 {#分区移除}

注：分区的移除可直接使用**virt-resize**工具完成，本章节仅演示**virt-rescue**工具的用法

分区移除涉及到分区操作，务必谨慎，确保硬盘镜像已备份。分区移除分为末尾分区移除和非末尾分区移除两种情况，末尾分区移除比较简单，不做介绍，这里只考虑非末尾分区的移除。比如说硬盘镜像中不再需要用到swap功能，于是乎需要将中间的swap分区移除。

使用**virt-rescue**工具操作该镜像：

```bash
export LIBGUESTFS_BACKEND=direct

virt-rescue -a centos.qcow2

><rescue>lsblk
NAME                        MAJ:MIN RM SIZE RO TYPE MOUNTPOINT

sda                           8:0    0  20G  0 disk

|-sda1                        8:1    0   1G  0 part

|-sda2                        8:2    0   2G  0 part

`-sda3                        8:3    0  17G  0 part

  `-centos_hikvisionos-root 252:0    0  17G  0 lvm

sdb                           8:16   0   4G  0 disk /
```

归档第三个分区，并删除lvm相关配置（防止分区同步报错）：

```
><rescue>mount /dev/centos_hikvisionos/root /sysroot/
><rescue> tar -cf rootfs.tgz /sysroot/
><rescue>umount /sysroot/
><rescue>lvremove centos_hikvisionos root
><rescue>vgremove centos_hikvisionos
```

删除第二、三分区，使用所有剩余空间重建为第二分区：

```
><rescue>fdisk /dev/sda
Command (m for help): p

...

Device     Boot   Start      End  Sectors Size Id Type

/dev/sda1  *       2048  2099199  2097152   1G 83 Linux

/dev/sda2       2099200  6293503  4194304   2G 82 Linux swap / Solaris

/dev/sda3       6293504 41943039 35649536  17G 83 Linux

d

d

n

Command (m for help): p

...

Device     Boot   Start      End  Sectors Size Id Type

/dev/sda1  *       2048  2099199  2097152   1G 83 Linux

/dev/sda2       2099200 41943039 39843840  19G 83 Linux

Command (m for help): w

The partition table has been altered.
```

重建文件系统：

shell

```
><rescue>pvcreate /dev/sda2
><rescue> vgcreate centos_hikvisionos /dev/sda2
><rescue>lvcreate --name root -l 100%FREE centos_hikvisionos
><rescue>mkfs.xfs /dev/centos_hikvisionos/root
><rescue>mount /dev/centos_hikvisionos/root /sysroot/
><rescue>tar -xf rootfs.tgz -C /
```

注意如果涉及到根分区或boot分区的改动，那么需要更新grub的启动配置：

shell

```
#
 这里省略guestmount并chroot的操作
#
 如果boot分区被改动，需要执行这一步将grub安装到该硬盘镜像中
#
 grub2-install /dev/sda
grub2-mkconfig -o /boot/grub2/grub.cfg

```

随后退出交互式shell，使用该镜像作为系统盘创建一台云服务器，调试该镜像是否能够正常启动。

### 分区扩容/缩容/删除 {#分区扩容缩容删除}

**virt-resize**工具可以用于硬盘镜像的分区操作，该工具在操作分区的同时也会考虑到其上的文件系统，较为方便。

shell

```
#
 创建一个空镜像，大小为10G，作为输出镜像
qemu-img create -f qcow2 -o preallocation=metadata test-new.qcow2 10G

#
 将sda2分区扩展至最大（9G）
virt-resize --expand /dev/sda2 test.qcow2 test-new.qcow2

#
 将sda2分区扩展至最大，与此同时sda1分区扩展200M
virt-resize --resize /dev/sda1=+200M --expand /dev/sda2 test.qcow2 test-new.qcow2

#
 将sda2分区扩展至最大，并将逻辑卷与文件系统扩容至最大
virt-resize --expand /dev/sda2 --LV-expand /dev/centos/root test.qcow2 test-new.qcow2

#
 缩容分区，注意缩容前需要事先缩容文件系统或pv，否则会报错
virt-resize --shrink /dev/sda2 test.qcow2 test-new.qcow2

virt-resize --resize /dev/sda1=-200M test.qcow2 test-new.qcow2

#
 删除分区
virt-resize --delete /dev/sda2 test.qcow2 test-new.qcow2

```

### 镜像空洞去除并压缩 {#镜像空洞去除并压缩}

**virt-sparsify**工具可以去除硬盘镜像的空洞并压缩。经测试，一个大小为4G的普通centos镜像能够压缩到700M，压缩比率较高，推荐使用，不过据工具说明，可能存在风险，目前尚未遇到有文件系统损坏的情况。

shell

```
qemu-img info hikos.raw|grep virtual

#
 virtual size: 6 GiB (6443696128 bytes)
#
 创建一个新的同样大小的镜像test.qcow2
qemu-img create -f qcow2 -o preallocation=metadata test.qcow2 6G

#
 使用virt-sparsify去除镜像空洞并压缩
virt-sparsify hikos.raw --compress --convert qcow2 test.qcow2

```

## 参考文档 {#参考文档}

[guestfish\(1\) - Linux man page \(die.net\)](https://linux.die.net/man/1/guestfish)

[KVM镜像管理利器-guestfish使用详解\_xiaoli110的技术博客\_51CTO博客](https://blog.51cto.com/xiaoli110/1568307)

