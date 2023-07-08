本文只讨论硬件的cache一致性机制，所以对软件来说是透明的。

![](https://pic4.zhimg.com/80/v2-64c71b57249012223f52085dea3a369f_720w.webp)

  


首先来看Cache和内存保持一致性的两种写入方式

**write through和write back**

CPU把数据写入 Cache 之后，内存与 Cache中 对应的数据就不一致了，所以要在一定的时机要把 Cache 中的数据同步到内存中。

根据写操作后同步到内存的时机，Cache和内存同步的方法可分为write back和write through。

![](https://pic4.zhimg.com/80/v2-6ca5dfb327d17645c699eff06ac128eb_720w.webp)

  


**write through**

CPU向cache写入数据时，同时也写入memory，使cache和memory的数据保持一致。

优点是简单，缺点是每次都要访问memory，速度比较慢。但是读数据时还是能够享受Cache带来的快速优点的。

**write back**

CPU向cache写入数据时，只是把更新的cache区标记一下（cache line 被标为dirty），并不同步写入memory。

只是在cache区要被刷入新的数据时，才更新memory。

优点是CPU执行的效率提高，缺点是实现起来技术比较复杂。

其中write back可以减少不必要的内存写入，减轻总线压力。现在大部分场景下，cache多采用write back的方式，本文的介绍都是基于write back的方式。

**单核一致性**

首先我们看一下单处理器情况下Cache和主存之间如何保持一致性。

读Cache:

![](https://pic1.zhimg.com/80/v2-1e3c3289d28d4443b57e420c1a7be2cc_720w.webp)

  


![](https://pic3.zhimg.com/80/v2-bc0bf2757e61d5f8611353d066dd2a42_720w.webp)

  


写Cache:

![](https://pic2.zhimg.com/80/v2-34645164ff0fb332fc85c1a258436061_720w.webp)

  


![](https://pic4.zhimg.com/80/v2-f503b1e44ad41c5cc35c5c0d4860a2e3_720w.webp)

  


如果是多处理器呢？

**多处理器的一致性问题**

举个例子吧，内存0x48处数据为0x20，处理器0和1都从0x48处读取内存数据到自己的Cache line中。

然后处理器0写Cache把0x48数据更新为0x10，处理器1读0x48自己Cache命中，返回了0x20。

出现两个处理器读到的内存数据不一致了！

![](https://pic4.zhimg.com/80/v2-9cddf8fb53fe959ea6a6cce8240d750b_720w.webp)

  


那么多处理器如何解决缓存一致性问题呢？

**多处理器的一致性方法**

多处理器一般是采用基于总线监听机制的高速缓存一致性协议。包括写无效和写更新协议。另外还有基于目录的高速缓存一致性机制。

**总线监听（Bus snooping）**

总线监听（Bus snooping）机制由 Ravishankar 和 Goodman 在 1983 年提出。其工作原理是当一个CPU修改了cache块之后，此更改必须传播到所有拥有该Cache 块副本的Cache上。

![](https://pic2.zhimg.com/80/v2-9d7e8c4127ea0bd9de0a9de08f4a55f5_720w.webp)

  


所有的监听者会监视总线上的所有数据广播。如果总线上出现修改共享Cache块的事件，所有监听者会检查自己的Cache是否缓存有共享Cache块的副本。

如果缓存有该共享Cache块的副本，则监听者执行操作以确保缓存一致性。

这些操作可以是刷新或失效缓存块，根据缓存一致性协议更改缓存块状态。

**两类总线监听协议**

根据管理本地Cache块副本的方式，有两类总线监听协议：

写更新（Write-update）

写无效（Write-invalidate）。

**写更新（Write-update）**

当处理器写入Cache块时，其他Cache监听到后把自己Cache中的数据副本进行更新。该方法通过总线向所有缓存广播写入数据。它比写无效协议产生更大的总线流量，所有这种方式不常见。Dragon和firefly属于这一类协议。

![](https://pic2.zhimg.com/80/v2-9a5848983a479359c68b73e97dcc7ff1_720w.webp)

  


**写无效（Write-invalidate）**

这是最常用的监听协议。当处理器写入Cache块时，其他Cache监听到后把自己Cache中的数据副本标记为无效状态。这样处理器只能读取和写入数据的一个副本，其他缓存中的副本都是无效的。

写直通无效协议、写一次协议、MSI、MESI、MOSI、MOESI、MESIF都属于写无效这一类协议。

![](https://pic4.zhimg.com/80/v2-a4be28792d00941e0c1cf289cb83fdcb_720w.webp)

  


下面以最为常用的MESI协议为例子分析写无效协议

**MESI**

MESI协议又叫Illinois协议，MESI，"M", "E", "S", "I"这4个字母代表了一个cache line的四种状态，分别是Modified,Exclusive,Shared和Invalid。

* Modified \(M\)
 

cache line只被当前cache所有，并且是dirty的。

* Exclusive \(E\)
 

cache line仅存在于当前缓存中，并且是clean的。

* Shared \(S\)
 

cache line在其他Cache中也存在并且都是clean的。

* Invalid \(I\)
 

cache line无效，即没有被任何Cache加载。

有一个著名的状态标记图：

![](https://pic1.zhimg.com/80/v2-923f1168474e7945f014790f041efac4_720w.webp)

  


这个状态标记图什么意思呢？

对同一个Cache line，

我标记它为是M时，你只能标记为I

我标记它为是E时，你只能标记为I

我标记它为是S时，你只能标记为S或I

我标记它为是I时，你能标记为MESI

MESI有一个状态机：

![](https://pic1.zhimg.com/80/v2-82b7637dd37250434df5664a7f7f24b8_720w.webp)

  


这个状态机什么意思呢？它显示了一种状态在出现什么Event时转换成哪一种状态，自己状态转换过程中要向总线上广播什么消息\(这些消息会被其他Cache监听到\)

下面的表是对这个状态机的详细说明:

![](https://pic2.zhimg.com/80/v2-9e99ba5d2b53689390c8bd0b8b847d69_720w.webp)

  


举个例子：

某Cache上一个cache line的现在状态是Shared。

如果本地CPU对它Read hit，那它状态还是Shared。

如果本地CPU对它Write hit，那它的状态变为Modified，并在总线上广播它Invalidate。

如果监听到总线上的Read消息，那它的状态还是Shared。

如果监听到总线上的Invalidate消息，那它的状态变为Invalidate。

其他的状态转换也是类似的处理。

**总线监听的优缺点**

如果有足够的带宽，总线监听比基于目录的一致性机制更快，因为所有事务都是直接被所有处理器看到。

总线侦听的缺点是可扩展性有限。频繁监听缓存会导致与处理器的访问竞争，从而增加缓存访问时间和功耗。每个请求都必须广播到系统中的所有节点。这意味着总线带宽必须随着系统变大而增长。由于总线侦听不能很好地扩展，较大的缓存一致性NUMA系统倾向于使用基于目录的一致性协议。

**基于目录（Directory-based）**

基于目录的一致性方法中，缓存的Cache块副本信息被保存在称为目录的结构中。当处理器写入Cache块时，不会向所有Cache广播请求，而是先查询目录以检索具有该副本的Cache，再发送到特定的处理器。与总线监听相比，目录方法可以大量节省总线流量。

在NUMA系统中，通常选择基于目录（directory-based）的方式来维护Cache的一致性。

