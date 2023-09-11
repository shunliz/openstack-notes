# Ceph优化系列（二）：Ceph主要配置参数详

## 概述 {#概述}

Ceph的配置参数很多，从网上也能搜索到一大批的调优参数，但这些参数为什么这么设置？设置为这样是否合理？解释的并不多  
本文从当前我们的ceph.conf文件入手，解释其中的每一项配置，做为以后参数调优和新人学习的依据；

## 参数详解 {#参数详解}

### 1，一些固定配置参数 {#1，一些固定配置参数}

\| \| fsid = 6d529c3d-5745-4fa5-be5f-3962a8e8687cmon\_initial\_members = mon1, mon2, mon3mon\_host = 10.10.40.67,10.10.40.68,10.10.40.69 \|

以上通常是通过ceph-deploy生成的，都是ceph monitor相关的参数，不用修改；

### 2，网络配置参数 {#2，网络配置参数}

\| \| public\_network = 10.10.40.0/24    默认值 ""cluster\_network = 10.10.41.0/24  默认值 "" \|

public network：monitor与osd，client与monitor，client与osd通信的网络，最好配置为带宽较高的万兆网络；  
cluster network：OSD之间通信的网络，一般配置为带宽较高的万兆网络；

**参考：**  
[http://docs.ceph.com/docs/master/rados/configuration/network-config-ref/](http://docs.ceph.com/docs/master/rados/configuration/network-config-ref/)

### 3，pool size配置参数 {#3，pool-size配置参数}

\|  \| osd\_pool\_default\_size = 3       默认值 3osd\_pool\_default\_min\_size = 1  默认值 0 // 0 means no specific default; ceph will use size-size/2 \|

这两个是创建ceph pool的时候的默认size参数，一般配置为3和1，3副本能足够保证数据的可靠性；

### 4，认证配置参数 {#4，认证配置参数}

\| \| auth\_service\_required = none    默认值 "cephx"auth\_client\_required = none     默认值 "cephx, none"auth\_cluster\_required = none   默认值 "cephx" \|

以上是Ceph authentication的配置参数，默认值为开启ceph认证；  
在内部使用的ceph集群中一般配置为none，即不使用认证，这样能适当加快ceph集群访问速度；

### 5，osd down out配置参数 {#5，osd-down-out配置参数}

\| \| mon\_osd\_down\_out\_interval = 864000  默认值 300 // secondsmon\_osd\_min\_down\_reporters = 2      默认值 2mon\_osd\_report\_timeout = 900        默认值 900osd\_heartbeat\_interval = 15         默认值 6osd\_heartbeat\_grace = 60            默认值 20 \|

`mon_osd_down_out_interval`：ceph标记一个osd为down and out的最大时间间隔  
`mon_osd_min_down_reporters`：mon标记一个osd为down的最小reporters个数（报告该osd为down的其他osd为一个reporter）  
`mon_osd_report_timeout`：mon标记一个osd为down的最长等待时间  
`osd_heartbeat_interval`：osd发送heartbeat给其他osd的间隔时间（同一PG之间的osd才会有heartbeat）  
`osd_heartbeat_grace`：osd报告其他osd为down的最大时间间隔，grace调大，也有副作用，如果某个osd异常退出，等待其他osd上报的时间必须为grace，在这段时间段内，这个osd负责的pg的io会hang住，所以尽量不要将grace调的太大。

基于实际情况合理配置上述参数，能减少或及时发现osd变为down（降低IO hang住的时间和概率），延长osd变为down and out的时间（防止网络抖动造成的数据recovery）；

**参考：**  
[http://docs.ceph.com/docs/master/rados/configuration/mon-osd-interaction/](http://docs.ceph.com/docs/master/rados/configuration/mon-osd-interaction/)  
[http://blog.wjin.org/posts/ceph-osd-heartbeat.html](http://blog.wjin.org/posts/ceph-osd-heartbeat.html)

### 6，objecter配置参数 {#6，objecter配置参数}

\| \| objecter\_inflight\_ops = 10240               默认值 1024objecter\_inflight\_op\_bytes = 1048576000     默认值 100M \|

osd client端objecter的throttle配置，它的配置会影响librbd，RGW端的性能；

**配置建议：**  
调大这两个值

### 7，ceph rgw配置参数 {#7，ceph-rgw配置参数}

\| \| rgw\_frontends = "civetweb port=10080 num\_threads=2000"  默认值 "fastcgi, civetweb port=7480"rgw\_thread\_pool\_size = 512                              默认值 100rgw\_override\_bucket\_index\_max\_shards = 20               默认值 0rgw\_max\_chunk\_size = 1048576                            默认值 512 \* 1024rgw\_cache\_lru\_size = 1000000                            默认值 10000 // num of entries in rgw cachergw\_bucket\_default\_quota\_max\_objects = 10000000         默认值 -1 // number of objects allowedrgw\_dns\_name = object-storage.ffan.com                  默认值rgw\_swift\_url = [http://object-storage.ffan.com](http://object-storage.ffan.com)          默认值 \|

`rgw_frontends`：rgw的前端配置，一般配置为使用轻量级的civetweb；prot为访问rgw的端口，根据实际情况配置；num\_threads为civetweb的线程数；  
`rgw_thread_pool_size`：rgw前端web的线程数，与rgw\_frontends中的num\_threads含义一致，但num\_threads 优于`rgw_thread_pool_size`的配置，两个只需要配置一个即可；  
`rgw_override_bucket_index_max_shards`：rgw bucket index object的最大shards数，增大这个值能减少bucket index object的访问时间，但也会加大bucket的ls时间；  
`rgw_max_chunk_size`：rgw最大chunk size，针对大文件的对象存储场景可以把这个值调大；  
`rgw_cache_lru_size`：rgw的lru cache size，对于读较多的应用场景，调大这个值能加快rgw的响应速度；  
`rgw_bucket_default_quota_max_objects`：配合该参数限制一个bucket的最大objects个数；

**参考：**  
[http://docs.ceph.com/docs/jewel/install/install-ceph-gateway/](http://docs.ceph.com/docs/jewel/install/install-ceph-gateway/)  
[http://ceph-users.ceph.narkive.com/mdB90g7R/rgw-increase-the-first-chunk-size](http://ceph-users.ceph.narkive.com/mdB90g7R/rgw-increase-the-first-chunk-size)  
[https://access.redhat.com/solutions/2122231](https://access.redhat.com/solutions/2122231)

### 8，debug配置参数 {#8，debug配置参数}

\| \| debug\_lockdep = 0/0debug\_context = 0/0debug\_crush = 0/0debug\_buffer = 0/0debug\_timer = 0/0debug\_filer = 0/0debug\_objecter = 0/0debug\_rados = 0/0debug\_rbd = 0/0debug\_journaler = 0/0debug\_objectcatcher = 0/0debug\_client = 0/0debug\_osd = 0/0debug\_optracker = 0/0debug\_objclass = 0/0debug\_filestore = 0/0debug\_journal = 0/0debug\_ms = 0/0debug\_mon = 0/0debug\_monc = 0/0debug\_tp = 0/0debug\_auth = 0/0debug\_finisher = 0/0debug\_heartbeatmap = 0/0debug\_perfcounter = 0/0debug\_asok = 0/0debug\_throttle = 0/0debug\_paxos = 0/0debug\_rgw = 0/0 \|

关闭了所有的debug信息，能一定程度加快ceph集群速度，但也会丢失一些关键log，出问题的时候不好分析；

**参考：**  
[http://www.10tiao.com/html/362/201609/2654062487/1.html](http://www.10tiao.com/html/362/201609/2654062487/1.html)

### 9，osd op配置参数 {#9，osd-op配置参数}

\|  \| osd\_enable\_op\_tracker = false       默认值 trueosd\_num\_op\_tracker\_shard = 32       默认值 32osd\_op\_threads = 10                 默认值 2osd\_disk\_threads = 1                默认值 1osd\_op\_num\_shards = 32                 默认值 5osd\_op\_num\_threads\_per\_shard = 2    默认值 2 \|

`osd_enable_op_tracker`：追踪osd op状态的配置参数，默认为true；不建议关闭，关闭后osd的 slow\_request，ops\_in\_flight，historic\_ops 无法正常统计；

\|  \| \# ceph daemon /var/run/ceph/ceph-osd.0.asok dump\_ops\_in\_flightop\_tracker tracking is not enabled now, so no ops are tracked currently, even those get stuck.  Please enable "osd\_enable\_op\_tracker", and the tracker will start to track new ops received afterwards.\# ceph daemon /var/run/ceph/ceph-osd.0.asok dump\_historic\_opsop\_tracker tracking is not enabled now, so no ops are tracked currently, even those get stuck.  Please enable "osd\_enable\_op\_tracker", and the tracker will start to track new ops received afterwards. \|

打开op tracker后，若集群iops很高，`osd_num_op_tracker_shard`可以适当调大，因为每个shard都有个独立的mutex锁；

\| \| class OpTracker {...struct ShardedTrackingData {Mutex ops\_in\_flight\_lock\_sharded;xlist&lt;TrackedOp \*&gt; ops\_in\_flight\_sharded;explicit ShardedTrackingData\(string lock\_name\):ops\_in\_flight\_lock\_sharded\(lock\_name.c\_str\(\)\) {}};vector&lt;ShardedTrackingData\*&gt; sharded\_in\_flight\_list;uint32\_t num\_optracker\_shards;...}; \|

`osd_op_threads`：对应的work queue有`peering_wq`（osd peering请求），`recovery_gen_wq`（PG recovery请求）；  
`osd_disk_threads`：对应的work queue为`remove_wq`（PG remove请求）；  
`osd_op_num_shards`和`osd_op_num_threads_per_shard`：对应的thread pool为`osd_op_tp`，work queue为`op_shardedwq`；

处理的请求包括：

1. `OpRequestRef`
2. `PGSnapTrim`
3. `PGScrub`

调大`osd_op_num_shards`可以增大osd ops的处理线程数，增大并发性，提升OSD性能；

### 10，osd client message配置参数 {#10，osd-client-message配置参数}

\| \| osd\_client\_message\_size\_cap = 1048576000    默认值 500\*1024L\*1024L     // client data allowed in-memory \(in bytes\)osd\_client\_message\_cap = 10000              默认值 100     // num client messages allowed in-memory \|

这个是osd端收到client messages的capacity配置，配置大的话能提升osd的处理能力，但会占用较多的系统内存；

**配置建议：**  
服务器内存足够大的时候，适当增大这两个值

### 11，osd scrub配置参数 {#11，osd-scrub配置参数}

\|  \| osd\_scrub\_begin\_hour = 2                默认值 0osd\_scrub\_end\_hour = 6                   默认值 24// The time in seconds that scrubbing sleeps between two consecutive scrubsosd\_scrub\_sleep = 2                       默认值 0        // sleep between \[deep\]scrub opsosd\_scrub\_load\_threshold = 5            默认值 0.5// chunky scrub配置的最小/最大objects数，以下是默认值osd\_scrub\_chunk\_min = 5osd\_scrub\_chunk\_max = 25 \|

Ceph osd scrub是保证ceph数据一致性的机制，scrub以PG为单位，但每次scrub回获取PG lock，所以它可能会影响PG正常的IO；

Ceph后来引入了chunky的scrub模式，每次scrub只会选取PG的一部分objects，完成后释放PG lock，并把下一次的PG scrub加入队列；这样能很好的减少PG scrub时候占用PG lock的时间，避免过多影响PG正常的IO；

同理，引入的`osd_scrub_sleep`参数会让线程在每次scrub前释放PG lock，然后睡眠一段时间，也能很好的减少scrub对PG正常IO的影响；

**配置建议：**

* `osd_scrub_begin_hour`
  和
  `osd_scrub_end_hour`
  ：OSD Scrub的开始结束时间，根据具体业务指定；
* `osd_scrub_sleep`
  ：osd在每次执行scrub时的睡眠时间；有个bug跟这个配置有关，建议关闭；
* `osd_scrub_load_threshold`
  ：osd开启scrub的系统load阈值，根据系统的load average值配置该参数；
* `osd_scrub_chunk_min`
  和
  `osd_scrub_chunk_max`
  ：根据PG中object的个数配置；针对RGW全是小文件的情况，这两个值需要调大；

**参考：**  
[http://www.jianshu.com/p/ea2296e1555c](http://www.jianshu.com/p/ea2296e1555c)  
[http://tracker.ceph.com/issues/19497](http://tracker.ceph.com/issues/19497)

### 12，osd thread timeout配置参数 {#12，osd-thread-timeout配置参数}

\| \| osd\_op\_thread\_timeout = 580               默认值 15osd\_op\_thread\_suicide\_timeout = 600         默认值 150osd\_recovery\_thread\_timeout = 580           默认值 30osd\_recovery\_thread\_suicide\_timeout = 600   默认值 300 \|

`osd_op_thread_timeout`和`osd_op_thread_suicide_timeout`关联的work queue为：

* `op_shardedwq`
  * 关联的请求为：
    `OpRequestRef`
    ，
    `PGSnapTrim`
    ，
    `PGScrub`
* `peering_wq`
  * 关联的请求为：osd peering

`osd_recovery_thread_timeout`和`osd_recovery_thread_suicide_timeout`关联的work queue为：

* `recovery_wq`
  * 关联的请求为：PG recovery

Ceph的work queue都有个基类`WorkQueue_`，定义如下：

\|  \| /// Pool of threads that share work submitted to multiple work queues.class ThreadPool : public md\_config\_obs\_t {.../// Basic interface to a work queue used by the worker threads.struct WorkQueue\_ {string name;time\_t timeout\_interval, suicide\_interval;WorkQueue\_\(string n, time\_t ti, time\_t sti\): name\(n\), timeout\_interval\(ti\), suicide\_interval\(sti\){ }... \|

这里的`timeout_interval`和`suicide_interval`分别对应上面所述的配置`timeout`和`suicide_timeout`；  
当thread处理work queue中的一个请求时，会受到这两个timeout时间的限制：

* `timeout_interval`
  * 到时间后设置
    `m_unhealthy_workers+1`
* `suicide_interval`
  * 到时间后调用assert，OSD进程crush

对应的处理函数为：

\|  \| bool HeartbeatMap::\_check\(const heartbeat\_handle\_d \*h, const char \*who, time\_t now\){bool healthy = true;time\_t was;was = h-&gt;timeout.read\(\);if \(was && was &lt; now\) {ldout\(m\_cct, 1\) &lt;&lt; who &lt;&lt; " '" &lt;&lt; h-&gt;name &lt;&lt; "'"&lt;&lt; " had timed out after " &lt;&lt; h-&gt;grace &lt;&lt; dendl;healthy = false;}was = h-&gt;suicide\_timeout.read\(\);if \(was && was &lt; now\) {ldout\(m\_cct, 1\) &lt;&lt; who &lt;&lt; " '" &lt;&lt; h-&gt;name &lt;&lt; "'"&lt;&lt; " had suicide timed out after " &lt;&lt; h-&gt;suicide\_grace &lt;&lt; dendl;assert\(0 == "hit suicide timeout"\);}return healthy;} \|

当前仅有RGW添加了worker的perfcounter，所以也只有RGW可以通过perf dump查看total/unhealthy的worker信息：

\| \| \[root@ yangguanjun\]\# ceph daemon /var/run/ceph/ceph-client.rgw.rgwdaemon.asok perf dump \| grep worker"total\_workers": 32,"unhealthy\_workers": 0 \|

对应的配置项为：

\| \| OPTION\(rgw\_num\_async\_rados\_threads, OPT\_INT, 32\) // num of threads to use for async rados operations\`\`\` \*\*配置建议：\*\*- \`\*\_thread\_timeout\`：这个值配置越小越能及时发现处理慢的请求，所以不建议配置很大；特别是针对速度快的设备，建议调小该值；- \`\*\_thread\_suicide\_timeout\`：这个值配置小了会导致超时后的OSD crush，所以建议调大；特别是在对应的throttle调大后，更应该调大该值；\#\#\# 13，fielstore op thread配置参数\`\`\`shfilestore\_op\_threads = 10                   默认值 2filestore\_op\_thread\_timeout = 580           默认值 60filestore\_op\_thread\_suicide\_timeout = 600   默认值 180 \|

`filestore_op_threads`：对应的thread pool为`op_tp`，对应的work queue为`op_wq`；filestore的所有请求都经过op\_wq处理；  
增大该参数能提升filestore的处理能力，提升filestore的性能；配合filestore的throttle一起调整；

`filestore_op_thread_timeout`和`filestore_op_thread_suicide_timeout`关联的work queue为：`op_wq`

配置的含义与上一节中的`thread_timeout/thread_suicide_timeout`保持一致；

### 13，filestore merge/split配置参数 {#13，filestore-merge-split配置参数}

\| \| filestore\_merge\_threshold = -1       默认值 10filestore\_split\_multiple = 16000      默认值 2 \|

这两个参数是管理filestore的目录分裂/合并的，filestore的每个目录允许的最大文件数为：  
`filestore_split_multiple * abs(filestore_merge_threshold) * 16`

在RGW的小文件应用场景，会很容易达到默认配置的文件数（320），若在写的过程中触发了filestore的分裂，则会非常影响filestore的性能；

每次filestore的目录分裂，会依据如下规则分裂为多层目录，最底层16个子目录：  
例如PG 31.4C0, hash结尾是4C0，若该目录分裂，会分裂为`DIR_0/DIR_C/DIR_4/{DIR_0, DIR_F}`；

原始目录下的object会根据规则放到不同的子目录里，object的名称格式为:`*__head_xxxxX4C0_*`，分裂时候X是几，就放进子目录DIR\_X里。比如object：`*__head_xxxxA4C0_*`, 就放进子目录`DIR_0/DIR_C/DIR_4/DIR_A`里；

**解决办法：**

1. 增大merge/split配置参数的值，使单个目录容纳更多的文件；
2. `filestore_merge_threshold`
   配置为负数；这样会提前触发目录的预分裂，避免目录在某一时间段的集中分裂，详细机制没有调研；
3. 创建pool时指定
   `expected-num-objects`
   ；这样会依据目录分裂规则，在创建pool的时候就创建分裂的子目录，避免了目录分裂对filestore性能的影响；

**参考：**  
[http://docs.ceph.com/docs/master/rados/configuration/filestore-config-ref/](http://docs.ceph.com/docs/master/rados/configuration/filestore-config-ref/)  
[http://docs.ceph.com/docs/jewel/rados/operations/pools/\#create-a-pool](http://docs.ceph.com/docs/jewel/rados/operations/pools/#create-a-pool)  
[http://blog.csdn.net/for\_tech/article/details/51251936](http://blog.csdn.net/for_tech/article/details/51251936)  
[http://ivanjobs.github.io/page3/](http://ivanjobs.github.io/page3/)

### 14，filestore fd cache配置参数 {#14，filestore-fd-cache配置参数}

\|  \| filestore\_fd\_cache\_shards =  32     默认值 16     // FD number of shardsfilestore\_fd\_cache\_size = 32768     默认值 128  // FD lru size \|

filestore的fd cache是加速访问filestore里的file的，在非一次性写入的应用场景，增大配置可以很明显的提升filestore的性能；

### 15，filestore sync配置参数 {#15，filestore-sync配置参数}

\| \| filestore\_wbthrottle\_enable = false    默认值 true        SSD的时候建议关闭filestore\_min\_sync\_interval = 5           默认值 0.01 s     最小同步间隔秒数，sync fs的数据到disk，FileStore::sync\_entry\(\)filestore\_max\_sync\_interval = 10         默认值 5 s       最大同步间隔秒数，sync fs的数据到disk，FileStore::sync\_entry\(\)filestore\_commit\_timeout = 3000         默认值 600 s       FileStore::sync\_entry\(\) 里 new SyncEntryTimeout\(m\_filestore\_commit\_timeout\) \|

`filestore_wbthrottle_enable`的配置是关于filestore writeback throttle的，即我们说的filestore处理workqueue`op_wq`的数据量阈值；默认值是true，开启后XFS相关的配置参数有：

\| \| OPTION\(filestore\_wbthrottle\_xfs\_bytes\_start\_flusher, OPT\_U64, 41943040\)OPTION\(filestore\_wbthrottle\_xfs\_bytes\_hard\_limit, OPT\_U64, 419430400\)OPTION\(filestore\_wbthrottle\_xfs\_ios\_start\_flusher, OPT\_U64, 500\)OPTION\(filestore\_wbthrottle\_xfs\_ios\_hard\_limit, OPT\_U64, 5000\)OPTION\(filestore\_wbthrottle\_xfs\_inodes\_start\_flusher, OPT\_U64, 500\)OPTION\(filestore\_wbthrottle\_xfs\_inodes\_hard\_limit, OPT\_U64, 5000\) \|

若使用普通HDD，可以保持其为true；针对SSD，建议将其关闭，不开启writeback throttle；

`filestore_min_sync_interval`和`filestore_max_sync_interval`是配置filestore flush outstanding IO到disk的时间间隔的；增大配置可以让系统做尽可能多的IO merge，减少filestore写磁盘的压力，但也会增大page cache占用内存的开销，增大数据丢失的可能性；

`filestore_commit_timeout`是配置filestore sync entry到disk的超时时间，在filestore压力很大时，调大这个值能尽量避免IO超时导致OSD crush；

### 16，filestore throttle配置参数 {#16，filestore-throttle配置参数}

\| \| filestore\_expected\_throughput\_bytes =  536870912       默认值 200MB    /// Expected filestore throughput in B/sfilestore\_expected\_throughput\_ops = 2500                默认值 200      /// Expected filestore throughput in ops/sfilestore\_queue\_max\_bytes= 1048576000                   默认值 100MBfilestore\_queue\_max\_ops = 5000                          默认值 50/// Use above to inject delays intended to keep the op queue between low and highfilestore\_queue\_low\_threshhold = 0.6                    默认值 0.3filestore\_queue\_high\_threshhold = 0.9                   默认值 0.9filestore\_queue\_high\_delay\_multiple = 2                    默认值 0    /// Filestore high delay multiple.  Defaults to 0 \(disabled\)filestore\_queue\_max\_delay\_multiple = 10                 默认值 0    /// Filestore max delay multiple.  Defaults to 0 \(disabled\) \|

在jewel版本里，引入了dynamic throttle，来平滑普通throttle带来的长尾效应问题；

一般在使用普通磁盘时，之前的throttle机制即可很好的工作，所以这里默认`filestore_queue_high_delay_multiple`和`filestore_queue_max_delay_multiple`都为0；

针对高速磁盘，需要在部署之前，通过小工具`ceph_smalliobenchfs`来测试下，获取合适的配置参数；

\| \| BackoffThrottle的介绍如下：/\*\*\* BackoffThrottle\*\* Creates a throttle which gradually induces delays when get\(\) is called\* based on params low\_threshhold, high\_threshhold, expected\_throughput,\* high\_multiple, and max\_multiple.\*\* In \[0, low\_threshhold\), we want no delay.\*\* In \[low\_threshhold, high\_threshhold\), delays should be injected based\* on a line from 0 at low\_threshhold to\* high\_multiple \* \(1/expected\_throughput\) at high\_threshhold.\*\* In \[high\_threshhold, 1\), we want delays injected based on a line from\* \(high\_multiple \* \(1/expected\_throughput\)\) at high\_threshhold to\* \(high\_multiple \* \(1/expected\_throughput\)\) +\* \(max\_multiple \* \(1/expected\_throughput\)\) at 1.\*\* Let the current throttle ratio \(current/max\) be r, low\_threshhold be l,\* high\_threshhold be h, high\_delay \(high\_multiple / expected\_throughput\) be e,\* and max\_delay \(max\_muliple / expected\_throughput\) be m.\*\* delay = 0, r \in \[0, l\)\* delay = \(r - l\) \* \(e / \(h - l\)\), r \in \[l, h\)\* delay = h + \(r - h\)\(\(m - e\)/\(1 - h\)\)\*/ \|

**参考：**  
[http://docs.ceph.com/docs/jewel/dev/osd\_internals/osd\_throttles/](http://docs.ceph.com/docs/jewel/dev/osd_internals/osd_throttles/)  
[http://blog.wjin.org/posts/ceph-dynamic-throttle.html](http://blog.wjin.org/posts/ceph-dynamic-throttle.html)  
[https://github.com/ceph/ceph/blob/master/src/doc/dynamic-throttle.txt](https://github.com/ceph/ceph/blob/master/src/doc/dynamic-throttle.txt)  
Ceph BackoffThrottle分析

### 17，filestore finisher threads配置参数 {#17，filestore-finisher-threads配置参数}

\| \| filestore\_ondisk\_finisher\_threads = 2   默认值 1filestore\_apply\_finisher\_threads = 2   默认值 1 \|

这两个参数定义filestore commit/apply的finisher处理线程数，默认都为1，任何IO commit/apply完成后，都需要经过对应的ondisk/apply finisher thread处理；

在使用普通HDD时，磁盘性能是瓶颈，单个finisher thread就能处理好；  
但在使用高速磁盘的时候，IO完成比较快，单个finisher thread不能处理这么多的IO commit/apply reply，它会成为瓶颈；所以在jewel版本里引入了finisher thread pool的配置，这里一般配置为2即可；

### 18，journal配置参数 {#18，journal配置参数}

\| \| journal\_max\_write\_bytes=1048576000          默认值 10M    journal\_max\_write\_entries=5000                默认值 100journal\_throttle\_high\_multiple = 2          默认值 0    /// Multiple over expected at high\_threshhold. Defaults to 0 \(disabled\).journal\_throttle\_max\_multiple = 10          默认值 0    /// Multiple over expected at max.  Defaults to 0 \(disabled\)./// Target range for journal fullnessOPTION\(journal\_throttle\_low\_threshhold, OPT\_DOUBLE, 0.6\)OPTION\(journal\_throttle\_high\_threshhold, OPT\_DOUBLE, 0.9\) \|

`journal_max_write_bytes`和`journal_max_write_entries`是journal一次write的数据量和entries限制；  
针对SSD分区做journal的情况，这两个值要增大，这样能增大journal的吞吐量；

`journal_throttle_high_multiple`和`journal_throttle_max_multiple`是`JournalThrottle`的配置参数，`JournalThrottle`是`BackoffThrottle`的封装类，所以`JournalThrottle`与我们在filestore throttle介绍的dynamic throttle工作原理一样；

\| \| int FileJournal::set\_throttle\_params\(\){stringstream ss;bool valid = throttle.set\_params\(g\_conf-&gt;journal\_throttle\_low\_threshhold,g\_conf-&gt;journal\_throttle\_high\_threshhold,g\_conf-&gt;filestore\_expected\_throughput\_bytes,g\_conf-&gt;journal\_throttle\_high\_multiple,g\_conf-&gt;journal\_throttle\_max\_multiple,header.max\_size - get\_top\(\),&ss\);...} \|

从上述代码中看出相关的配置参数有：

* `journal_throttle_low_threshhold`
* `journal_throttle_high_threshhold`
* `filestore_expected_throughput_bytes`

### 19，rbd cache配置参数 {#19，rbd-cache配置参数}

\| \| \[client\]rbd\_cache\_size = 134217728                  默认值 32M // cache size in bytesrbd\_cache\_max\_dirty = 100663296             默认值 24M // dirty limit in bytes - set to 0 for write-through cachingrbd\_cache\_target\_dirty = 67108864           默认值 16M // target dirty limit in bytesrbd\_cache\_writethrough\_until\_flush = true   默认值 true    // whether to make writeback caching writethrough until flush is called, to be sure the user of librbd will send flushs so that writeback is saferbd\_cache\_max\_dirty\_age = 5                 默认值 1.0     // seconds in cache before writeback starts \|

`rbd_cache_size`：client端每个rbd image的cache size，不需要太大，可以调整为64M，不然会比较占client端内存；  
参照默认值，根据`rbd_cache_size`的大小调整`rbd_cache_max_dirty`和`rbd_cache_target_dirty`；

* `rbd_cache_max_dirty`
  ：在writeback模式下cache的最大bytes数，默认是24MB；当该值为0时，表示使用writethrough模式；
* `rbd_cache_target_dirty`
  ：在writeback模式下cache向ceph集群写入的bytes阀值，默认16MB；注意该值一定要小于
  `rbd_cache_max_dirty`
  值

`rbd_cache_writethrough_until_flush`：在内核触发flush cache到ceph集群前rbd cache一直是writethrough模式，直到flush后rbd cache变成writeback模式；  
`rbd_cache_max_dirty_age`：标记OSDC端ObjectCacher中entry在cache中的最长时间；

## 参考 {#参考}

[https://my.oschina.net/linuxhunter/blog/541997](https://my.oschina.net/linuxhunter/blog/541997)







