# ceph逻辑结构

## Object

Object是Ceph的最小存储单元，大小可以自己定义通常为2M或4M，每个Object都包含了在集群范围内唯一的标识OID、二进制数据、以及由一组键值对组成的元数据。OID是由ino与ono生成，ino是文件的file id，用于全局标识每一个文件，ono则是分片的编号。无论上层应用的是RBD、CephFS还是RADOSGW，最终都会以Object的方式存储在OSD上。当RADOS接收到来向客户端的数据写请求时，它将接收到的数据转换为Object，然后OSD守护进程将数据写入OSD文件系统上的一个文件中。

![](/assets/storage-ceph-logicstructure1.png)

## Pool

Pool是Ceph中的一些object的逻辑分组，它只是一个逻辑概念，类似于LVM中的Volume Group，类似于一个命名空间。Pool由若干个PG组成，其属性包括：所有者和访问权限、Object副本数目、PG数目和CRUSH规则集合等。用户可以对不同的Pool设置相关的优化策略，比如PG副本数、数据清洗次数、数据块及Object的Size等。当把数据写人到一个 Pool 时，首先会在CRUSH Map找到与Pool对应的规则集，这个规则集描述了Pool的副本数信息。在部署Ceph集群时会默认创建data、metadata和rbd Pool。

1）Ceph中的Pool有两种类型

Replicated pool：通过生成Object的多份拷贝来确保在部分OSD丢失的情况下数据不丢失。这种类型的pool需要更多的裸存储空间，但是它支持所有的pool操作。

Erasure-coded pool：纠错码型pool，类似于software RAID。在这种pool中，每个数据对象都被存放在K+M个数据块中：对象被分成K个数据块和M个编码块；pool的大小被定义成K+M块，每个块存储在一个OSD 中；块的顺序号作为object的属性保存在对象中。可见，这种pool用更少的空间实现存储，即节约空间；纠删码实现了高速的计算，但有2个缺点，一个是速度慢，一个是只支持对象的部分操作。

2）Pool提供如下能力

Resilience（弹力）：即在确保数据不丢失的情况允许一定的OSD失败，这个数目取决于对象的副本数。对Replicated pool来说，Ceph中默认的副本数是2，这意味着除了对象自身外，它还有一个另外的备份，这个副本数由参数osd\_pool\_default\_size指定。

Placement Groups（放置组）：Ceph使用PG来组织对象，这是因为对象可能成千上万，因此一个一个对象来组织的成本是非常高的。PG的值会影响Ceph集群的行为和数据的持久性。通过osd\_pool\_default\_pg\_num设置pool中的PG 数目，默认为32，推荐的配置是每个OSD大概100个PG。

CRUSH Rules（CRUSH 规则）：数据映射的策略，由参数osd\_pool\_default\_crush\_rule指定，用户可以自定义策略来灵活地设置object存放的区域。

Snapshots（快照）：你可以对pool做快照

Set Ownership：设置pool的owner的用户ID

3）Pool相关的操作如下

创建pool

ceph osd pool create {poolname} {pg-num} \[{pgp-num}\] \[replicated\] \[crush-ruleset-name\] \[expected-num-objects\]

ceph osd pool create {poolname} {pg-num} {pgp-num} erasure \[erasure-code-profile\] \[crush-ruleset-name\] \[expected\_num\_objects

配置Pool配额

ceph osd pool set-quota {poolname} \[max\_objects {obj-count}\] \[max\_

删除Pool

ceph osd pool delete {poolname} \[{poolname} --yes-i-really-really

重命名Pool

ceph osd pool rename {current-pool-name} {new-pool-name}

给Pool做快照

ceph osd pool mksnap {poolname} {snap-name}

删除Pool快照

ceph osd pool rmsnap {poolname} {snap-name}

配置Pool相关参数

ceph osd pool set {poolname} {key} {value}

获取Pool参数的值

ceph osd pool get {poolname} {key}

配置对象副本数目

ceph osd pool set {poolname} size {num-replicas}

获取对象副本数目

ceph osd dump \| grep 'replicated size'

## PG（Placement GROUP）

### 1.3.1 PG的概念

在Ceph集群中有着成千上万的objects，对于一个TB规模以上的集群，每个objects的默认大小为4MB，以三副本为例，整个集群可以存储的objects就有1TB/4MB/3=87381个，这样直接以objects为粒度进行数据管理的代价过于昂贵且不灵活。

PG是一些Objects的集合，这些objects组成一个group，放在某些OSD上（place），组合起来就是Placement Group。将objects以PG为单位进行管理，有以下好处：

集群中的PG数目经过规划因为严格可控，使得基于PG可以精准控制单个OSD乃至整个节点的资源消耗，如CPU、内存、网络带宽等

因为集群中的PG数目远小于objects数目，并且PG数目和每个PG的身份相对固定，以PG为单位进行数据备份策略和数据同步、迁移等，相较于直接以对象为单位而言，难度更小且更加灵活

PG中涉及到的一些概念如下：

Epoch：PG map的版本号，由OSD monitor生成，它是一个单调递增的序列。Epoch变化意味着OSD map发生了变化，需要通过一定的策略扩散到所有客户端和位于服务端的OSD。

Peering：归属于同一个PG所有的PG实例就本PG所存储的全部对象及对象相关的元数据状态进行协商并最终达到一致的过程。

Acting set：支持一个PG的所有OSD的有序列表，其中第一个OSD是主OSD，其余为从OSD。Acting set是CRUSH算法分配的，指当前或者曾在某个interval负责承载对应PG的PG实例。

Up set：某一个PG map历史版本的acting set。在大多数情况下，acting set 和 up set 是一致的，除非出现了 pg\_temp。

Current Interval or Past Interval：若干个连续的版本号，这些版本中acting set和up set保持不变。

PG temp：Peering过程中，如果当前interval通过CRUSH计算得到的Up Set不合理，那么可以通知OSD Monitor设置PG temp来显示的指定一些仍然具有相对完备PG信息的OSD加入Acting set，在Ceph 正在往主 OSD 回填数据时，这个主OSD是不能提供数据服务的，使得Acting set中的OSD在完成Peering之后能够临时处理客户端发起的读写请求，以尽可能的减少业务的中断时间。举个例子，现在acting set是\[0,1,2\]，出现了一点事情后，它变为\[3,1,2\]，此时osd.3还是空的因此它无法提供数据服务因此它还需要等待backfilling过程结束，因此它会向MON申请一个临时的set比如\[1,2,3\]，此时将由 osd.1提供数据服务。回填过程结束后，该临时set会被丢弃，重新由osd.3提供服务。

Backfilling：recovery一种特殊场景，指Peering完成后，如果基于当前权威日志无法对Up Set当中的某些PG实例实施增量同步，则通过完全拷贝当前Primary所有对象的方式进行全量同步

Authoriative History：权威日志是Peering过程中执行增量数据同步的依据，通过交换信息并基于一定的规则从所有PG实例中选举产生

Primary OSD：在acting set中的首个 OSD，负责接收客户端写入数据；默认情况下，提供数据读服务，但是该行为可以被修改。它还负责peering过程，以及在需要的时候申请PG temp。

Replica OSD：在acting set中的除了第一个以外的其余OSD。

Stray OSD：PG实例所在的OSD已经不是acting set 中了，但是还没有被告知去删除数据的OSD。如果对应的PG已经完成Peering，并且处于Active+Clean状态，那么这个实例稍后将被删除；如果对应的PG尚未完成Peering，那么这个PG实例仍有可能转化为Replica

### 1.3.2 PG的特点

#### 1）基本特点

PG确定了pool中的对象和OSD之间的映射关系。一个object只会存在于一个PG中，但是多个object可以在同一个PG内。

Pool的PG数目是创建pool时候指定的，其值与OSD的总数的关系密切。当Ceph集群扩展OSD 增多时，根据需要可以增加pool的PG数目。

Object的副本数目，是在创建Pool时指定的，决定了每个PG会在几个OSD上保存对象。如果一个Replicated pool的size为2，它会包含指定数目的PG，每个PG使用两个OSD，其中第一个为主OSD，其它的为从OSD。不同的PG可能会共享一个OSD。

Ceph引入PG的目的主要是为了减少直接将对象映射到OSD的复杂度。

PG也是Ceph集群做清理（scrubbing）的基本单位，也就是说数据清理是一个一个PG来操作的。

PG和OSD之间的映射关系由CRUSH决定，而它做决定的依据是CRUSH rules。CRUSH将所有的OSD组织成一个分层结构，该结构能区分故障域，该结构中每个节点都是一个CRUSH bucket。

#### 2）PG和OSD的关系是动态的

一开始在PG被创建的时候，MON根据CRUSH算法计算出PG所在的OSD，这是它们之间的初始关系。

Ceph集群中OSD的状态是不断变化的，它会在如下状态之间做切换：

up：守护进程运行中，能够提供IO服务

down：守护进程不在运行，无法提供IO服务

in：包含数据

out：不包含数据

部分PG和OSD的关系会随着OSD状态的变化而发生变化

当新的OSD被加入集群后，已有OSD上部分PG将可能被挪到新OSD上，此时PG和OSD 的关系会发生改变。

当已有的某OSD down了并变为out后，其上的PG会被挪到其它已有的OSD上

但是大部分的PG和OSD的关系将会保持不变，在状态变化时Ceph尽可能只挪动最少的数据

客户端根据Cluster map以及CRUSH Rulese 使用CRUSH算法查找出某个PG所在的OSD列表

#### 1.3.3 PG的创建过程

PG的创建是由monitor节点发起，形成请求message发送给osd，在osd上创建pg

1. MON节点上有PGMonitotor，它发现有pool被创建后，判断该pool是否有PG。如果有PG，则一一判断这些PG是否已经存在，如果不存在，则开始下面的创建PG的过程。
2. 创建过程的开始，设置PG状态为 Creating，并将它加入待创建PG队列 creating\_pgs，等待被处理。
3. 开始处理后，使用CRUSH算法根据当前的OSD map找出来up/acting set，加入PG map 中的队列 creating\_pgs\_by\_osd。
4. 队列处理函数将该OSD上需要创建的PG合并，生成消息MOSDPGCreate，通过消息通道发给OSD。
5. OSD收到消息字为 MSG\_OSD\_PG\_CREATE的消息，得到消息中待创建的PG信息，判断类型，并获取该PG的其它OSD，加入队列 creating\_pgs，再创建具体的 PG。
6. PG被创建出来以后，开始Peering过程。

#### 1.3.4 PG值的确定

创建pool时需要确定其PG的数目，在pool被创建后也可以调整该数字，PG值有以下影响因素：

**数据的持久性**：假如pool的size为 3，表明每个PG会将数据存放在3个OSD上。当一个OSD down了后，一定间隔后将开始recovery过程，recovery和需要被恢复的数据的数量有关系，如果该 OSD 上的 PG 过多，则花的时间将越长，风险将越大。在recovery结束前有部分PG的数据将只有两个副本，如果此时再有一个OSD down了，那么将有一部分PG的数据只有一个副本。recovery 过程继续，如果再出现第三个OSD down了，那么可能会出现部分数据丢失。可见，每个OSD上的PG数目不宜过大，否则会降低数据的持久性。这也就要求在添加OSD后，PG的数目在需要的时候也需要相应增加。

**数据的均匀分布性**：CRUSH算法会伪随机地保证PG被选中来存放客户端的数据，它还会尽可能地保证所有的PG均匀分布在所有的OSD上。比方说，有10个OSD，但是只有一个size为3的pool，它只有一个PG，那么10个 OSD 中将只有三个OSD被用到。但是CURSH算法在计算的时候不会考虑到OSD上已有数据的大小。比方说，100万个4K对象共4G均匀地分布在10个OSD上的1000个PG内，那么每个OSD上大概有400M 数据。再加进来一个400M的对象，那么有三块OSD上将有400M + 400M = 800M的数据，而其它七块OSD上只有400M数据。

**资源消耗**：PG作为一个逻辑实体，它需要消耗一定的资源，包括内存、CPU和带宽。太多PG的话，则占用资源会过多。

**清理时间**：Ceph的清理工作是以PG为单位进行的。如果一个PG内的数据太多，则其清理时间会很长。

那如何确定一个 Pool 中有多少 PG？Ceph 不会自己计算，而是给出了一些参考原则，让 Ceph 用户自己计算：

| OSD数目 | 1~5 | 5~10 | 10~50 | &gt;50 |
| :--- | :--- | :--- | :--- | :--- |
| PG数目 | 建议为128 | 建议为512 | 建议为4096 | 使用 pgcalc 工具 |

### 1.3.5 PG的状态迁移

PG外部状态的变化最终通过其内部状态机进行驱动，集群状态的变化最终转换为一系列状态机中的事件，驱动状态机在不同状态之间进行跳转和执行处理，从而实现我们所希望的功能和行为。PG 的所有的状态是一个类似树形的结构，每个状态可能存在子状态，子状态还可能存在子状态，如下图所示：![](/assets/storage-ceph-logicstructure2.png)

* Creating创建中：PG正在被创建
* Peering对等互联：表示一个过程，该过程中一个PG的所有OSD都需要互相通信来PG的对象及其元数据的状态达成一致。处于该状态的P不能响应IO请求。Peering的过程其实就是pg状态从初始状态然后到active+clean的变化过程。一个OSD启动之后，上面的pg开始工作，状态为initial，这时进行比对所有osd上的pglog和pg\_info，对pg的所有信息进行同步，选举primary osd和replica osd，peering过程结束，然后把peering的结果交给recovering，由recovering过程进行数据的恢复工作。
* Active活动的：Peering过程完成后，PG的状态就是active的。此状态下，在主次OSD上的PG数据都是可用的。
* Clean洁净的：此状态下，主次OSD都已经被peered了，每个副本都就绪了。
* Down：PG掉线了，因为存放其某些关键数据（比如pglog和pginfo，它们也是保存在OSD上）的副本 down了。
* Degraded降级的：某个OSD被发现停止服务了后，Ceph MON将该OSD上的所有PG的状态设置为degraded，此时该OSD的peer OSD会继续提供数据服务。这时会有两种结果：一是它会重新起来（比如重启机器时），需要再经过peering过程再到clean状态，而且Ceph会发起recovery过程，使该OSD上过期的数据被恢复到最新状态；二是OSD的down状态持续300秒后其状态被设置为out，Ceph会选择其它的OSD加入acting set，并启动回填（backfilling）数据到新OSD的过程，使PG副本数恢复到规定的数目。
* Recovering恢复中：一个OSD down后，其上面的PG的内容的版本会比其它OSD上的PG副本的版本落后。在它重启之后，Ceph会启动recovery过程来使其数据得到更新。
* Backfilling回填中：一个新OSD加入集群后，Ceph会尝试级将部分其它OSD上的PG挪到该新OSD上，此过程被称为回填。与recovery相比，回填（backfill）是在零数据的情况下做全量拷贝，而恢复（recovery）是在已有数据的基础上做增量恢复。
* Remapped重映射：每当PG的acting set改变后，就会发生从旧 acting set到新acting set 的数据迁移。此过程结束前，旧acting set中的主OSD将继续提供服务。一旦该过程结束，Ceph将使用新acting set中的主OSD来提供服务。
* Stale过期的：OSD每隔0.5秒向MON报告其状态。如果因为任何原因，主OSD报告状态失败了，或者其它OSD已经报告其主 OSD down 了，Ceph MON 将会将它们的PG标记为stale状态。



