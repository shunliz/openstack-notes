**LVM几个基本概念**

\*物理存储介质（PhysicalStorageMedia）：指系统的物理存储设备：磁盘，如：/dev/hda、/dev/sda等，是[存储系统](http://baike.baidu.com/view/51839.htm)最底层的[存储单元](http://baike.baidu.com/view/1223079.htm)。

\*物理卷（Physical Volume，PV）：可以在上面建立卷组的媒介，可以是硬盘分区，也可以是硬盘本身或者回环文件（loopback file），它是LVM的基本存储逻辑块。物理卷包括一个特殊的header，其余部分被切割为一块块物理区域（physical extents）。

\*物理块（Physical Extent，PE）：每一个物理卷PV被划分为称为PE（Physical Extents）的[基本单元](http://baike.baidu.com/view/693012.htm)，具有唯一编号的PE是可以被LVM寻址的最小单元。PE的大小是可配置的，默认为4MB。所以物理卷（PV）由大小等同的基本单元PE组成。

\*卷组（Volume Group，VG）：将一组物理卷收集为一个管理单元，类似于非LVM系统中的物理磁盘，其由一个或多个物理卷PV组成。可以在卷组上创建一个或多个LV（逻辑卷）。

\*逻辑卷（Logical Volume，LV）：虚拟分区，由物理区域（physical extents）组成，逻辑卷建立在卷组VG之上。在逻辑卷LV之上可以建立文件系统（比如/home或者/usr等）。

\*逻辑块（Logical Extent，LE）：逻辑卷LV也被划分为可被[寻址](http://baike.baidu.com/view/1303626.htm)的基本单位，称为LE。在同一个卷组中，LE的大小和PE是相同的，并且一一对应。

LVM抽象模型

* ![](https://images0.cnblogs.com/blog/697113/201412/111401427904725.jpg)
  ![](https://images0.cnblogs.com/blog/697113/201412/111405543687283.jpg)
* ![](https://images0.cnblogs.com/blog/697113/201412/111747184935692.jpg)

### 优点 {#blogTitle0}

比起正常的硬盘分区管理，LVM更富于弹性：

* 使用卷组\(VG\)，使众多硬盘空间看起来像一个大硬盘。
* 使用逻辑卷（LV），可以创建跨越众多硬盘空间的分区。
* 可以创建小的逻辑卷（LV），在空间不足时再动态调整它的大小。
* 在调整逻辑卷（LV）大小时可以不用考虑逻辑卷在硬盘上的位置，不用担心没有可用的连续空间。It does not depend on the position of the LV within VG, there is no need to ensure surrounding available space.
* 可以在线（online）对逻辑卷（LV）和卷组（VG）进行创建、删除、调整大小等操作。LVM上的文件系统也需要重新调整大小，某些文件系统也支持这样的在线操作。
* 无需重新启动服务，就可以将服务中用到的逻辑卷（LV）在线（online）/动态（live）迁移至别的硬盘上。
* 允许创建快照，可以保存文件系统的备份，同时使服务的下线时间（downtime）降低到最小。

这些优点使得LVM对服务器的管理非常有用，对于桌面系统管理的帮助则没有那么显著，你需要根据实际情况进行取舍。

### 缺点： {#blogTitle1}

* 只能在Linux上使用。对于其他操作系统（如FreeBSD, Windows等），尚未有官方支持。
* 在系统设置时需要更复杂的额外步骤。
* 假如你使用的是
  [btrfs](https://wiki.archlinux.org/index.php/Btrfs)
  文件系统，那么它所提供的子卷（subvolume）实际上已经时一层可动态调整的存储层，此时再用LVM就显得多余了。



