**该文同时发表在盛大游戏G云微信公众号，粘贴于此，方便各位查阅**

Ceph，相信很多IT朋友都听过。因为搭上了Openstack的顺风车，Ceph火了，而且越来越火。然而要用好Ceph却也不是件易事，在QQ群里就经常听到有初学者抱怨Ceph性能太烂，不好用。事实果真如此吗！如果你采用Ceph的默认配置来运行你的Ceph集群，性能自然不能如人意。俗话说，玉不琢,不成器；Ceph也有它的脾性，经过良好配置优化的Ceph性能还是不错的。下文简单分享下，盛大游戏G云在Ceph优化上的一些实际经验，如有错误之处，欢迎指正。

> 下文的Ceph配置参数摘自Ceph Hammer 0.94.1 版本

# Ceph配置参数优化 {#ceph配置参数优化}

首先来看看Ceph客户端与服务端的交互组件图：



Ceph是一个统一的可扩展的分布式存储，提供`Object`,`Block`及`file system`三种访问接口；它们都通过底层的`librados`与后端的`OSD`交互；`OSD`是Ceph的对象存储单元，实现数据的存储功能。其内部包含众多模块，模块之间通过队列交换消息，相互协作共同完成io的处理；典型模块有：网络模块`Messenger`，数据处理模块`Filestore`,日志处理模块`FileJournal`等。

面对众多的模块，Ceph也提供了丰富的配置选项，初步统计Ceph有上千个配置参数，要配置好这么多参数，难度有多大可想而知。G云内部主要使用`Ceph Block Storage`,即：`Ceph rbd`；下文的配置参数优化也就限于`rbd`客户端（`librbd`）和`OSD`端。  
下面先来看看客户端的优化

## `rbd`客户端配置优化 {#rbd客户端配置优化}

当Ceph作为虚拟机块存储使用时，`Qemu`是通过`librbd`，这个客户端库与Ceph集群交互的；与其相关的配置参数基本都以`rbd_`为前缀。可以通过如下命令获取`librbd`的当前配置：

```
//path/to/socket指向某个osd的admin socket文件
#
>
 ceph --admin-daemon {path/to/socket} config show | grep rbd

```

下面对其中的某些配置参数进行详细说明：

* `rbd cache`
  : 是否使能缓存，默认情况下开启。
* `rbd cache size`
  :最大的缓存大小，默认32MB
* `rbd cache max dirty`
  :缓存中脏数据的最大值，用来控制回写，不能超过
  `rbd cache size`
  ，默认24MB
* `rbd cache target dirty`
  :开始执行回写的脏数据大小，不能超过
  `rbd cache max dirty`
  ，默认16MB
* `rbd cache max dirty age`
  : 缓存中单个脏数据的最大缓存时间，避免因为未达到回写要求脏数据长时间存在缓存中，默认1s

> 点评：开启Cache能显著提升顺序io的读写性能，缓存越大性能越好；如果容许一定的数据丢失，建议开启。

* `rbd cache max dirty object`
  :最大的
  `Object`
  对象数，默认为0，表示通过
  `rbd cache size`
  计算得到，
  `librbd`
  默认以4MB为单位对磁盘Image进行逻辑切分，每个
  `chunk`
  对象抽象为一个
  `Object`
  ；
  `librbd`
  中以
  `Object`
  为单位来管理缓存，增大该值可以提升性能。

> 点评：在ceph-0.94.1中该值较小，建议按照ceph-0.94.4版本中的计算公式增大该值，如下：
>
> obj = MIN\(2000, MAX\(10, cct-&gt;\_conf-&gt;rbd\_cache\_size / 100 / sizeof\(ObjectCacher::Object\)\)\);
>
> 我配置的时候取 sizeof\(ObjectCacher::Object\) = 128, 128是我基于代码估算的`Object`对象大小

* `rbd cache writethrough until flush`
  :默认为true，该选项是为了兼容linux-2.6.32之前的virtio驱动，避免因为不发送flush请求，数据不回写；设置该参数后，
  `librbd`
  会以
  `writethrough`
  的方式执行io，直到收到第一个flush请求，才切换为
  `writeback`
  方式。

> 点评：如果您的Linux客户机使用的是2.6.32之前的内核建议设置为true，否则可以直接关闭。

* `rbd cache block writes upfront`
  ：是否开启同步io，默认false,开启后
  `librbd`
  要收到
  `Ceph OSD`
  的应答才返回。

> 点评： 开启后，性能最差，但是最安全。

* `rbd readahead trigger requests`
  : 触发预读的连续请求数，默认为10
* `rbd readahead max bytes`
  : 一次预读请求的最大io大小，默认512KB，为0则表示关闭预读
* `rbd readahead disable after bytes`
  : 预读缓存的最大数据量，默认为50MB，超过阀值后，
  `librbd`
  会关闭预读功能，由Guest OS处理预读（防止重复缓存）；如果为0，则表示不限制缓存。

> 点评：如果是顺序读io为主，建议开启

* `objecter inflight ops`
  : 客户端流控，允许的最大未发送io请求数，超过阀值会堵塞应用io，为0表示不受限
* `objecter inflight op bytes`
  :客户端流控，允许的最大未发送脏数据，超过阀值会堵塞应用io，为0表示不受限

> 点评：提供了简单的客户端流控功能，防止网络拥塞；在宿主网络成为瓶颈的情况下，`rbd cache`中可能充斥着大量`处于发送`状态的io，这又会反过来影响io性能。没有特殊需要的话，不需要修改该值；当然如果带宽足够的话，可以根据需要调高该值

* `rbd ssd cache`
  : 是否开启磁盘缓存，默认开启
* `rbd ssd cache size`
  :缓存的最大大小，默认10G
* `rbd ssd cache max dirty`
  :缓存中脏数据的最大值，用来控制回写，不能超过
  `rbd ssd cache size`
  ，默认7.5G
* `rbd ssd cache target dirty`
  :开始执行回写的脏数据大小，不能超过
  `rbd cache max dirty`
  ，默认5G
* `rbd ssd chunk order`
  :缓存文件分片大小，默认64KB = 2^16
* `rbd ssd cache path`
  :缓存文件所在的路径

> 点评：这是盛大游戏G云自己开发的带掉电保护的rbd cache,前面四个参数与前述`rbd cache *`含义相似，`rdb ssd chunk size`定义缓存文件的分片大小，是缓存文件的最小分配/回收单位，分片大小直接影响缓存文件的使用效率；在运行过程中`librbd`也会基于io大小动态计算分片大小，并在合适的时候应用到缓存文件.

上面就是盛大游戏G云在使用Ceph rbd过程中，在客户端所做优化的一些经验，如有错误还请多多批评指正，也欢迎各位补充。继续来看看`OSD`的调优

## `OSD`配置优化 {#osd配置优化}

`Ceph OSD`端包含了众多配置参数，所有的配置参数定义在`src/common/config_opts.h`文件中，当然也可以通过命令查看集群的当前配置参数：

```
#
>
 ceph --admin-daemon {path/to/socket} config show

```

由于能力有限，下面仅分析常见的几个配置参数：

* `osd op threads`
  :处理peering等请求的线程数
* `osd disk threads`
  :处理snap trim，replica trim及scrub等的线程数
* `filestore op threads`
  ：io线程数

> 点评：相对来说线程数越多，并发度越高，性能越好；然如果线程太多，频繁的线程切换也会影响性能；所以在设置线程数时，需要全面考虑节点CPU性能，OSD个数以及存储介质性质等。通常前两个参数设置一个较小的值，而最后一个参数设置一个较大的值，以加快io处理速度。在发生peering等异常时可以动态的调整相关的值。

* `filestore op thread timeout`
  :io线程超时告警时间
* `filestore op thread suicide timeout`
  ：io线程自杀时间，当一个线程长时间没有响应，Ceph会终止该线程，并导致OSD进程退出

> 点评： 如果io线程出现超时，应结合相关工具/命令分析\(如：`ceph osd perf`\)，是否`OSD`所在的磁盘存在瓶颈，或者介质故障等

* `ms dispatch throttle bytes`
  :
  `Messenger`
  流控，控制
  `DispatcherQueue`
  队列深度，0表示不受限

> 点评：`Messenger`处在`OSD`的最前端，其性能直接影响io处理速度。在Hammer版本中，io请求添加到`OSD::op_shardedwq`才会返回，其他一些请求会直接添加到`DispatchQueue`中；为避免`Messenger`成为瓶颈，可以将该值设大点

* `osd_op_num_shards`
  :默认为5，
  `OSD::op_shardedwq`
  中存储io的队列个数
* `osd_op_num_threads_per_shard`
  : 默认为2，为
  `OSD::op_shardedwq`
  中每个队列分配的io分发线程数

> 点评：作用于`OSD::op_shardedwq`的总线程数为：`osd_op_num_shards`\*`osd_op_num_threads_per_shard`, 默认为10；io请求经由`Messenger`进入，添加到`OSD::op_shardedwq`后，由上述分发线程发送给后端的`filestore`处理。视前端网络（如：10Gbps）和后端介质性能\(如：SSD\)，可考虑调高该值。

* `filestore_queue_max_ops`
  :控制filestore中队列的深度，最大未完成io数
* `filestore_queue_max_bytes`
  :控制filestore中队列的深度，最大未完成io数据量
* `filestore_queue_commiting_max_ops`
  :如果
  `OSD`
  后端的文件系统支持检查点，则
  `filestore_queue_max_ops`
  +
  `filestore_queue_commiting_max_ops`
  作为filestore中队列的最大深度，表示最大未完成io数
* `filestore_queue_commiting_max_bytes`
  :如果
  `OSD`
  后端的文件系统支持检查点，则
  `filestore_queue_max_bytes`
  +
  `filestore_queue_commiting_max_bytes`
  作为filestore中队列的最大深度，表示最大未完成io数据量

> 点评： filestore收到前述分发线程递交的io后，其处理过程首先会受到filestore队列深度的影响；如果队列中未完成io超过设置阀值，请求将会堵塞。所以调高该值是不个很明智的选择。

* `journal_queue_max_ops`
  : 控制FileJournal中队列的深度，最大未完成日志io数
* `journal_queue_max_bytes`
  : 控制FileJournal中队列的深度，最大未完成日志io数据量

> 点评： filestore收到前述分发线程递交的io后，还会受到FileJournal队列的影响；如果队列中未完成io超过设置阀值，请求同样会堵塞；通常，调高该值是个不错的选择；另外，采用独立的日志盘， io性能也会有不少的提升

* `journal_max_write_entries`
  : FileJournal一次异步日志io最大能处理的条目数
* `journal_max_write_bytes`
  : FileJournal一次异步日志io最大能处理的数据量

> 点评： 这两个参数控制日志异步io每次能处理的最大io数，通常要根据日志文件所在磁盘的性能来设置一个合理值

* `filestore_wbthrottle_enable`
  :默认为true，控制
  `OSD`
  后端文件系统刷新
* `filestore_wbthrottle_*_bytes_start_flusher`
  :xfs/btrfs文件系统开始执行回刷的脏数据
* `filestore_wbthrottle_*_bytes_hard_limit`
  : xfs/btrfs文件系统允许的最大脏数据，用来控制filestore的io操作
* `filestore_wbthrottle_*_ios_start_flusher`
  :xfs/btrfs文件系统开始执行回刷的io请求数
* `filestore_wbthrottle_*_ios_hard_limit`
  :xfs/btrfs文件系统允许的最大未完成io请求数，用来控制filestore的io操作
* `filestore_wbthrottle_*_inodes_start_flusher`
  :xfs/btrfs文件系统开始执行回刷的对象数
* `filestore_wbthrottle_*_inodes_hard_limit`
  :xfs/btrfs文件系统允许的最大脏对象，用来控制filestore的io操作

> 点评：\*\_start\_flusher参数定义了刷新xfs/btrfs文件系统脏数据阀值，以将磁盘缓存更新到磁盘；\*\_hard\_limit参数会影响filestore的io操作，堵塞`filestore op thread`线程。所以设置一个较大的值性能会有提升

* `filestore_fd_cache_size`
  : 对象文件句柄缓存大小
* `filestore_fd_cache_shards`
  : 对象文件句柄缓存分片个数

> 点评：缓存文件句柄能加快文件的访问速度，个人建议缓存所有的文件句柄，当然请记得调高系统的句柄限制，以免句柄耗尽

* `filestore_fiemap`
  : 开启稀疏读写特性

> 点评： 开启该特性，有助于加快克隆和恢复速度

* `filestore_merge_threshold`
  : pg子目录合并的最小文件数
* `filestore_split_multiple`
  : pg子目录分裂乘数，默认为2

> 点评：这两个参数控制pg目录的合并与分裂，当目录下的文件数小于`filestore_merge_threshold`时，该目录的对象文件会被合并到父目录；如果目录的文件数大于`filestore_merge_threshold*16*filestore_split_multiple`,该目录会分裂成两个子目录。设置合理的值可以加快对象文件的索引速度

* `filestore_omap_header_cache_size`
  : 扩展属性头缓存

> 点评： 缓存对象的扩展属性\_`Header`对象，减少对后端leveldb数据库的访问，提升查找性能

纯属抛砖引玉，结合个人的实践经验，上文给出了Ceph的一些配置参数设置建议，希望能给大家一些思路。Ceph的配置还是比较复杂的，是一项系统工程，各个参数的配置需要综合考虑各节点的网络情况，CPU性能，磁盘性能等等因素，欢迎大家留言讨论！:-\)

