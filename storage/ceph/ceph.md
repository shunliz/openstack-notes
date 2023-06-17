ceph-deploy时可以指定内网和外网两个网段

# 1 Ceph项目简述

        Ceph最早起源于Sage就读博士期间的工作、成果于2004年发表，并随后贡献给开源社区。经过多年的发展之后，已得到众多云计算和存储厂商的支持，成为应用最广泛的开源分布式存储平台。



        Ceph根据场景可分为对象存储、块设备存储和文件存储。Ceph相比其它分布式存储技术，其优势点在于：它不单是存储，同时还充分利用了存储节点上的计算能力，在存储每一个数据时，都会通过计算得出该数据存储的位置，尽量将数据分布均衡。同时，由于采用了CRUSH、HASH等算法，使得它不存在传统的单点故障，且随着规模的扩大，性能并不会受到影响。



## 1.1 Ceph的特点

高性能



摒弃了传统的集中式存储元数据寻址的方案，采用CRUSH算法，数据分布均衡，并行度高。

考虑了容灾域的隔离，能够实现各类负载的副本放置规则，例如跨机房、机架感知等。

能够支持上千个存储节点的规模，支持TB到PB级的数据。

高可用性



副本数可以灵活控制。

支持故障域分隔，数据强一致性。

多种故障场景自动进行修复自愈。

没有单点故障，自动管理。

高可扩展性



去中心化。

扩展灵活。

随着节点增加而线性增长。

特性丰富



支持三种存储接口：块存储、文件存储、对象存储。

支持自定义接口，支持多种语言驱动。

当然，Ceph也存在一些缺点：

去中心化的分布式解决方案，需要提前做好规划设计，对技术团队的要求能力比较高。

Ceph扩容时，由于其数据分布均衡的特性，会导致整个存储系统性能的下降。

# 2 Ceph的主要架构

![](/assets/storage-ceph-ceph1.png)

* Ceph的最底层是RADOS（分布式对象存储系统），它具有可靠、智能、分布式等特性，实现高可靠、高可拓展、高性能、高自动化等功能，并最终存储用户数据。RADOS系统主要由两部分组成，分别是OSD和Monitor。
* RADOS之上是LIBRADOS，LIBRADOS是一个库，它允许应用程序通过访问该库来与RADOS系统进行交互，支持多种编程语言，比如C、C++、Python等。
* 基于LIBRADOS层开发的有三种接口，分别是RADOSGW、librbd和MDS。
* RADOSGW是一套基于当前流行的RESTFUL协议的网关，支持对象存储，兼容S3和Swift。
* librbd提供分布式的块存储设备接口，支持块存储。
* MDS提供兼容POSIX的文件系统，支持文件存储。





Ceph采用crush算法，在大规模集群下，实现数据的快速、准确存放，同时能够在硬件故障或扩展硬件设备时，做到尽可能小的数据迁移，其原理如下：



1. 当用户要将数据存储到Ceph集群时，数据先被分割成多个object，\(每个object一个object id，大小可设置，默认是4MB），object是Ceph存储的最小存储单元。
2. 由于object的数量很多，为了有效减少了Object到OSD的索引表、降低元数据的复杂度，使得写入和读取更加灵活，引入了pg\(Placement Group \)：PG用来管理object，每个object通过Hash，映射到某个pg中，一个pg可以包含多个object。
3. Pg再通过CRUSH计算，映射到osd中。如果是三副本的，则每个pg都会映射到三个osd，保证了数据的冗余。

![](/assets/storage-ceph-cephflow1.png)

![](/assets/storage-ceph-cephflow1.png)

![](/assets/storage-ceph-cephflow1.png)

![](/assets/storage-ceph-cephflow1.png)

![](/assets/storage-ceph-cephflow1.png)

![](/assets/storage-ceph-cephflow1.png)

![](/assets/storage-ceph-cephflow1.png)

![](/assets/storage-ceph-cephflow1.png)





