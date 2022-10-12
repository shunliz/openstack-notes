创建

1、给数据资源池添加tier层

ceph osd tier add data\_pool cache\_pool --force-nonempty

2、设置tier模式为writeback

ceph osd tier cache-mode cache\_pool writeback

3、设置tier层overlay

ceph osd tier set-overlay data\_pool cache\_pool

4、设置过滤器

ceph osd pool set cache\_pool hit\_set\_type bloom

5、设置hit count数量

ceph osd pool set cache\_pool hit\_set\_count 4

6、设置target\_max\_bytes

ceph osd pool set cache\_pool target\_max\_bytes

7、设置第一条水线

ceph osd pool set cache\_pool cache\_target\_dirty\_ratio 0.4

8、设置第二条水线

ceph osd pool set cache\_pool cache\_target\_dirty\_high\_ratio 0.6

9、设置第三条水线

ceph osd pool set cache\_pool cache\_target\_full\_ratio 0.8



缓存池原理

缓存分层特性也是在Ceph的Firefly版中正式发布的，这也是Ceph的Firefly版本中被谈论最多的一个特性。缓存分层是在更快的磁盘（通常是SSD），上创建一个Ceph池。这个缓存池应放置在一个常规的复制池或erasure池的前端，这样所有的客户端I/O操作都首先由缓存池处理。之后，再将数据写回到现有的数据池中。客户端能够在缓存池上享受高性能，而它们的数据显而易见最终是被写入到常规池中的。

![](/assets/storage-ceph-cephcachetier1.png)

一般来说，缓存层构建在昂贵/速度更快的SSD磁盘上，这样才能为客户提供更好的I/O性能。在缓存池后端通常是存储层，它由复制或者erasure类型的HDD组成。在这种类型的设置中，客户端将I/O请求提交到缓存池，不管它是一个读或写操作，它的请求都能够立即获得响应。速度更快的缓存层为客户端请求提供服务。一段时间后，缓存层将所有数据写回备用的存储层，以便它可以缓存来自客户端的新请求。在缓存层和存储层之间的数据迁移都是自动触发且对客户端是透明的。缓存分层能以两种模式进行配置。



writeback模式：当Ceph的缓存分层配置为writeback模式时，Ceph客户端将数据写到缓存层类型的池，也就是速度更快的池，因此能够立即接收写入确认。基于你为缓存层设置的flushing/evicting策略，数据将从缓存层迁移到存储层，并最终由缓存分层代理将其从缓存层中删除。处理来自客户端的读操作时，首先由缓存分层代理将数据从存储层迁移到缓存层，然后再把它提供给客户。直到数据变得不再活跃或成为冷数据，否则它将一直保留在缓存层中。



read-only模式：当Ceph的缓存分层配置为read-only模式时，它只适用于处理客户端的读操作。客户端的写操作不涉及缓存分层，所有的客户端写都在存储层上完成。在处理来自客户端的读操作时，缓存分层代理将请求的数据从存储层复制到缓存层。基于你为缓存层配置的策略，不活跃的对象将会从缓存层中删除。这种方法非常适合多个客户端需要读取大量类似数据的场景。



缓存层是在速度更快的物理磁盘（通常是SSD）上实现的，它在使用HDD构建的速度较慢的常规池前部署一个快速的缓存层。在本节中，我们将创建两个独立的池（一个缓存池和一个常规），分别用作缓存层和存储层。



![](/assets/storage-ceph-cephcachetier2.png)



理论与实践相结合

1、下面开始配置以cache作为sata-pool的前端高速缓冲池。



1）新建缓冲池，其中，cache作为sata-pool的前端高速缓冲池。



\# ceph osd pool create storage 64



pool ‘storage’ created



\# ceph osd pool create cache 64



pool ‘cache’ created



2）设定缓冲池读写策略为写回模式。



ceph osd tier cache-mode cache writeback



3）把缓存层挂接到后端存储池上



\# ceph osd tier add storage cache



pool ‘cache’ is now \(or already was\) a tierof ‘storage’



4）将客户端流量指向到缓存存储池



\# ceph osd tier set-overlay storage cache



overlay for ‘storage’ is now \(or alreadywas\) ‘cache’



2、调整Cache tier配置



1）设置缓存层hit\_set\_type使用bloom过滤器



\# ceph osd pool set cache hit\_set\_type bloom



set pool 27 hit\_set\_type to bloom



命令格式如下：



ceph osd pool set {cachepool} {key} {value}



关于Bloom-Filte算法原理可参见：



https://blog.csdn.net/jiaomeng/article/details/1495500



2）设置hit\_set\_count、hit\_set\_period



\# ceph osd pool set cache hit\_set\_count 1



set pool 27 hit\_set\_count to 1



\# ceph osd pool set cache hit\_set\_period3600



set pool 27 hit\_set\_period to 3600



\# ceph osd pool set cache target\_max\_bytes1000000000000



set pool 27 target\_max\_bytes to1000000000000



默认情况下缓冲池基于数据的修改时间来进行确定是否命中缓存，也可以设定热度数hit\_set\_count和热度周期hit\_set\_period，以及最大缓冲数据target\_max\_bytes。



hit\_set\_count 和 hit\_set\_period 选项分别定义了 HitSet 覆盖的时间区间、以及保留多少个这样的 HitSet，保留一段时间以来的访问记录，这样 Ceph 就能判断一客户端在一段时间内访问了某对象一次、还是多次（存活期与热度）。



3）设置min\_read\_recency\_for\_promote、min\_write\_recency\_for\_promote



\# ceph osd pool set cachemin\_read\_recency\_for\_promote 1



set pool 27 min\_read\_recency\_for\_promote to1



\# ceph osd pool set cachemin\_write\_recency\_for\_promote 1



set pool 27 min\_write\_recency\_for\_promote to 1



缓存池容量控制

先讲解个概念缓存池代理层两大主要操作



·刷写（flushing）：负责把已经被修改的对象写入到后端慢存储，但是对象依然在缓冲池。



·驱逐（evicting）：负责在缓冲池里销毁那些没有被修改的对象。



缓冲池代理层进行刷写和驱逐的操作，主要和缓冲池本身的容量有关。在缓冲池里，如果被修改的数据达到一个阈值（容量百分比），缓冲池代理就开始把这些数据刷写到后端慢存储。当缓冲池里被修改的数据达到40%时，则触发刷写动作。



\# ceph osd pool set cachecache\_target\_dirty\_ratio 0.4



当被修改的数据达到一个确定的阈值（容量百分比），刷写动作将会以高速运作。例如，当缓冲池里被修改数据达到60%时候，则高速刷写。



\# ceph osd pool set cachecache\_target\_dirty\_high\_ratio 0.6



缓冲池的代理将会触发驱逐操作，目的是释放缓冲区空间。例如，当缓冲池里的容量使用达到80%时候，则触发驱逐操作。



\# ceph osd pool set cachecache\_target\_full\_ratio 0.8



除了上面提及基于缓冲池的百分比来判断是否触发刷写和驱逐，还可以指定确定的数据对象数量或者确定的数据容量。对缓冲池设定最大的数据容量，来强制触发刷写和驱逐操作。



\# ceph osd pool set cache target\_max\_bytes1073741824



同时，也可以对缓冲池设定最大的对象数量。在默认情况下，RBD的默认对象大小为4MB，1GB容量包含256个4MB的对象，则可以设定：



\# ceph osd pool set cache target\_max\_objects 256



4）缓冲池的数据刷新问题在缓冲池里，对象有最短的刷写周期。若被修改的对象在缓冲池里超过最短周期，将会被刷写到慢存储池。



\# ceph osd pool set cachecache\_min\_flush\_age 600



注意：单位是分钟



设定对象最短的驱逐周期。



\# ceph osd pool set cachecache\_min\_evict\_age 1800



删除缓存层

删除readonly缓存



1）把缓存模式改为 none 即可禁用。



ceph osd tier cache-mode {cachepool} none



2）去除后端存储池的缓存池。



ceph osd tier remove {storagepool}{cachepool}



删除writeback缓存



1）把缓存模式改为 forward ，这样新的和更改过的对象将直接刷回到后端存储池



\# ceph osd tier cache-mode cache forward–yes-i-really-mean-it



set cache-mode for pool ‘cache’ to forward



2）确保缓存池已刷回，可能要等数分钟



\# rados ls -p cache



可以通过以下命令进行手动刷回



\# rados -p cache cache-flush-evict-all



3）取消流量指向缓存池



\# ceph osd tier remove-overlay storage



there is now \(or already was\) no overlayfor ‘storage’



4）剥离缓存池



\# ceph osd tier remove storage cache



pool ‘cache’ is now \(or already was\) not atier of ‘storage’



5）刷tier层数据

rados -p ram1 cache-flush-evict-all



