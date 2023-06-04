# OpenFlow

除了基于overlay网络的思想设计以外，OpenVSwitch的另一大特点就是基于OpenFlow。

传统的交换机，不论是硬件的，还是软件的，所具备的功能都是预先内置的，需要使用某个功能的时候，进行相应的配置即可。而OpenVSwitch通过OpenFlow实现了交换机的可编程。OpenFlow可以定义网络包在交换机中的处理流程（pipeline），因此支持OpenFlow的交换机，其功能不再是固定的，通过OpenFlow可以软件定义OpenVSwitch所具备的功能。

OpenFlow以多个Table串行工作的方式来处理网络数据包，如下图所示。

OpenFlow的灵活性是实现SDN必不可少的一部分，但是在一些实际场景中，因为涉及的功能多且复杂，相应的OpenFlow pipeline会变得很长。直观上来看，pipeline越长，处理一个网络包需要的时间也越长。这是OpenFlow从理论到实际的一个问题，OpenVSwitch为此尝试过很多优化。

对于一个Linux系统来说，可以分为用户空间（user space）和内核空间（kernel space），网络设备接入到内核空间。如果需要将数据传输到用户程序则需要通过内核空间将数据上送到用户空间，如果需要在网络设备之间转发数据，直接在内核空间就可以完成。

作为运行在x86服务器中的软件交换机，直观上来看，应该在内核空间来实现转发。因此，OpenVSwitch在最早期的时候，在Linux内核模块实现了所有的OpenFlow的处理。当时的OpenVSwitch内核模块，接收网络数据包，根据OpenFlow规则，一步步的Match，并根据Action修改网络数据包，最后从某个网络设备送出。

但是这种方式很快就被认为是不能实际应用的。首先，虽然在内核实现可以缩短网络数据包在操作系统的路径，但是在内核进行程序开发和更新也更加困难，以OpenVSwitch的更新速度，完全在内核实现将变得不切实际。其次，完全按照OpenFlow pipeline去处理网络包，势必要消耗大量CPU，进而降低网络性能。

因此，最新版本（2.x版本）的OpenVSwitch采用了一种很不一样的方式来避免这些问题。

## OpenFlow规则查找

OpenFlow规则基于Match-Action而实现。直观上看的话，Match实现更简单，只需要匹配就行。但是在实际中，为了让包的处理性能达到可用的程度，实现Match比实现Action要复杂的多。因为Match可以是2-4层协议Header中的任意field或者任意的metadata。假设有一条OpenFlow规则匹配了源MAC，目的MAC，源IP，目的IP，源端口，目的端口，那么对于所有需要经过OpenFlow处理网络数据包，都需要将自己的Header中这些filed与这条OpenFlow进行对比，以确保自己匹配或者不匹配这条OpenFlow规则。假设有10000条这样的OpenFlow规则，那么这个网络数据包最坏情况下需要与这10000条规则的Match做对比，才知道自己匹配或者不配。如果OpenFlow有200个这样的table，每个table都有10000条OpenFlow规则呢？所以，直接匹配是没有实际意义的。因为实际环境中，OpenFlow规则很容易达到数万条甚至数十万条。就算OpenVSwitch分了内核空间和用户空间程序，只有首包需要走OpenFlow pipeline，但是光是首包这样处理，也会大大降低网络性能，同时要消耗大量CPU。而OpenVSwitch作为运行在主机的软件交换机，应该尽量减少CPU的消耗，把CPU留给workload。

因此在用户空间，ovs-vswitchd基于TSS实现OpenFlow的匹配，以减少Match所需要的时间。以前面的OpenFlow规则为例，对于需要匹配的源MAC，目的MAC，源IP，目的IP，源端口，目的端口这些具体值，hash成一个散列值，这样10000条规则对应10000个散列值。对于任何需要经过OpenFlow处理的网络数据包，先将其源MAC，目的MAC，源IP，目的IP，源端口，目的端口hash成一个散列值，然后在OpenFlow规则对应的散列值中查找。根据hash的特性，查找时间复杂度为O\(1\)，就是说，不论有多少条OpenFlow规则，都能在固定时间确认匹配或者不匹配。在一个OpenFlow Table中，可能同时存在多种匹配类型，有的只匹配MAC地址，有的只匹配in\_port，对于不同的Match，OpenVSwitch会将其分类，每一类Match构成一个hash table。一个OpenFlow Table里有多少种不同的Match，就会有多少个hash table。当一个网络包需要经过一个OpenFlow Table处理时，网络数据包会在所有的hash table中进行查找，如果所有的hash table都不匹配，那么网络包就miss了，默认会drop掉；如果匹配了一个hash table，那么就按照hash table里具体那一条规则的action执行；如果有多个hash table匹配，那么就看优先级，优先级高的胜出。因此，在下发OpenFlow规则时，应当尽量将相同的Match的OpenFlow规则下发在一个Table中。这样可以减少一个Table中生成的hash table的数量。虽然这算是一个不强的限制，但是本身也符合实际，因为一个OpenFlow Table通常被指定实现某个网络功能，因此会包含大量相同Match的OpenFlow规则。

虽然查找算法很多，但是TSS具有以下优点，使其比其他算法更适合OpenVSwitch和SDN的场景：

O\(1\) 的更新时间。前面只看了查找时间，但是SDN控制器同时可能会频繁的更新OpenFlow规则，因此更新时间的性能也很重要。

hash算法对于任意的OpenFlow Match Field都适用，不论是4层协议的Header还是metadata，还是OpenVSwitch自己扩展的寄存器。

TSS对内存的消耗是线性的，也就是OpenFlow的规则数与TSS消耗的内存成正比。

实际使用中，在一个OpenFlow Table中查找OpenFlow规则的性能很重要，但是如何在多个Table中查找也很重要。一个实际环境中的OpenFlow table数，通常会在几十左右，这些table共同构成了OpenFlow的pipeline。导致Table数变多的因素有两个：第一是模块化的设计，使得每个table专注一个功能，随着网络功能的增多，对应的table也变多了；第二就是处理向量积的问题，例如匹配目的IP为100个离散的IP，且目的端口是离散的100个端口，这个时候，要么在一个table里面写10000（100\*100）条OpenFlow规则，要么在两个Table里面分别写100条规则，这种情况下，为了节省OpenFlow规则数，相应的table数就要增加。

因此，缩短OpenFlow pipeline，对于性能的提升也很重要。

**Microflow Caching**

如前一篇所述，OVS内核模块早期基于名为microflow的cache。microflow也是基于Match-Action而实现，但是它的Match包含了所有OpenFlow可能的匹配的值，也就是2-4层包头和一些metadata。如果用前面描述的TSS查找算法去实现microflow cache，只需要一个hash table即可，因为这里只有一种match。

只要microflow cache已经建立了，可以实现OVS内核模块的快速查找转发，因为只需要查找一次Hash table就可以完成转发。所以现在的问题是如何快速的建立microflow cache。microflow cache的建立是一个被动的过程，当一个网络数据包在microflow cache对应的hash table中查找失败的时候，会从OVS内核模块上送到ovs-vswitchd，由ovs-vswitchd通过OpenFlow pipeline处理之后，下发这个网络数据包对应的datapath actions，进而建立网络数据包到datapath actions的对应关系。所以cache建立的时间，由两部分组成：第一部分是网络包在用户态ovs-vswitchd进程走OpenFlow pipeline所需要的时间，这个在前一小节介绍过；第二部分是网络包上送的比例。

从整体上来说，越低的网络包上送比例，就能越快的建立cache，进而有越高的网络性能。而通过microflow cache实现的OVS内核模块，无疑每个不同传输层连接都需要将自己的首包上送到ovs-vswitchd。甚至同一个网络连接都需要多次上送网络包，例如因为网络多路径导致IP TTL的不同，也会导致microflow cache查找失败（miss）。对于大量的短连接来说，会有大量的首包上送。因此microflow cache在实际应用的时候并不能达到一个很好的性能。

**Megaflow Caching**

如果我们来看下面一条OpenFlow流表规则。

priority=200,ip,nw\_dst=10.0.0.0/16 actions=output:1

它的作用就是匹配所有目的IP是10.0.0.0/16的网络包，从网卡1送出。对于下面的传输层连接，对应的action都是一样的。

11.0.0.2:5742 -&gt; 10.0.0.10:3306

11.0.0.2:5743 -&gt; 10.0.0.10:3306

11.0.0.2:5744 -&gt; 10.0.0.11:3306

11.0.0.3:5742 -&gt; 10.0.0.10:3306

但是对应于microflow cache来说，就要有4条cache，需要上送4次ovs-vswitchd。但是实际上，如果在kernel datapath如果有下面一条cache，只匹配目的IP地址：

ip,nw\_dst=10.0.0.0/16 actions=output:1

那么数以亿计的传输层连接，可以在OVS内核模块，仅通过这条cache，完成转发。因为只有一条cache，所以也只有一次上送ovs-vswitchd。这样能大大减少内核态向用户态上送的次数，从而提升网络性能。这样一条非精确匹配的cache，被OpenVSwitch称为megaflow。

Megaflow能提升网络性能的前提是，大量的网络的连接可以共用一组datapath action。因为这样，在microflow cache中对应的Hashtable里面，就会有大量的key拥有相同的value，这些key本身存在合并的可能。

Megaflow cache更像是一个OpenFlow Table，因为它支持任意形式的packet field matching。类似的，OpenVSwitch也是通过TSS来实现megaflow cache，由于现在Match有多种类型，所以对应于megaflow cache最终会有多个hash table。OVS内核模块转发时，一个网络数据包需要在这多个hash table中进行查找。在OpenVSwitch中，megaflow对应的hash table数称为Mask。同时相比OpenFlow Table，megaflow cache有一些简化，一个是它没有priority，这样在TSS查找时，只要找到了就可以立刻终止查找（不用考虑更高优先级）；第二个，为了避免混淆，userspace只会下发非重合的megaflow cache，这样一个网络包只会跟一条megaflow cache匹配；第三，megaflow cache只有一个table，而不像OpenFlow是由多个table组成的pipeline。

OpenVSwitch提供命令打开或者关闭Megaflow cache。

ovs-appctl upcall/disable-megaflows

ovs-appctl upcall/enable-megaflows

**Microflow vs Megaflow**

应该说，Megaflow和Microflow各有优缺点。Microflow的优点是，只有一个Hash table，一旦cache建立，内核中只需要查找一次Hash table即可完成转发，缺点是每个不同的网络连接都需要一次内核空间上送用户空间，这有点多。Megaflow的优点是，合并了拥有相同的datapath action的网络连接所对应的cache，对于所有这些网络连接，只需要一次内核空间上送用户空间，缺点是cache建立以后，要查多次Hash table。如果每个Hash table命中的概率一样，那么平均需要查找 \(n+1\)/2 次hash table，最坏要查n个hash table。

选择Megaflow还是Microflow，就要看实际中究竟是内核空间上送用户空间的消耗大，还是在内核模块转发时，多查几次hash table代价大。对于长连接占多数的网络流量来说，Microflow占优势，对于短连接占多数的网络流量来说，Megaflow占优势。那在实现OVS内核模块时，究竟应该选用哪种cache模型。俗话说，小孩子才做选择，成年人全部都要。所以OVS内核模块采用Microflow+Megaflow的两级cache模式。

Microflow作为第一级cache，还是匹配所有OpenFlow可能的匹配的值，但是这时对应的Action不再是Datapath action，而是送到某个Megaflow的Hash table。当网络包送到kernel datapath时，会先在Microflow cache中查找，如果找到了，那么再接着送到相应的Megaflow hash table继续查找；如果没有找到，网络包会在kernel datapath的Megaflow cache中继续查找，如果找到了，会增加Microflow cache的记录，将后继类似的包直接指向相应的Megaflow Hash table；如果在Megaflow还是没有找到，那么就要上送ovs-vswitchd，通过OpenFlow pipeline生成Megaflow cache。

这样不论Megaflow cache有多少个hash table，对于长连接来说，只需要查找两次hash table即可完成转发，长连接的性能得到了保证。对于短连接来说，因为Megaflow cache的存在，也不用频繁的上送ovs-vswitchd，只需要在OVS内核模块多查几次hash table就行，短连接的性能也得到了保证。

这里似乎看起来没有问题，但是因为Megaflow现在是采用模糊匹配，究竟怎么生成Megaflow成了个问题。Megaflow应当尽量模糊，以匹配更多的网络连接，从而减少上送ovs-vswitchd的次数，下一篇看看OpenFlow的定义下，如何生成尽量模糊的Megaflow Cache。

Megaflow的规模越小，对应的hash table数量越少，相应的kernel datapath的转发性能更好。为了减少megaflow的规模，OpenVSwitch只有在有包上送到ovs-vswitchd之后，才会生成megaflow cache，也就是说，incoming packets驱动了megaflow cache的生成。这与早期的Microflow cache的生成方式一样。不同于早期的microflow cache的是，megaflow cache是一种模糊匹配，并且应该尽量模糊，因为这样可以匹配更多不同的网络连接，减少上送ovs-vswitchd的次数。但是又不能太模糊，因为这样会导致匹配出错。所以，怎么确定一条megaflow cache的匹配内容？

例如前面介绍的OpenFlow规则：

priority=200,ip,nw\_dst=10.0.0.0/16 actions=output:1

匹配了Ethernet Type（IP）和IP header中目的IP地址的前16bit。对于下面的网络连接对应的网络数据包：

11.0.0.2:5742 -&gt; 10.0.0.10:3306

上送ovs-vswitchd生成的megaflow cache如下，Megaflow会且仅会匹配在经过OpenFlow pipeline时，用到过的field（可以精确到field的某些bit，例如这里的目的IP前16bit）。

ip,nw\_dst=10.0.0.0/16 actions=output:1

这样，在其他类似网络连接到来时

11.0.0.2:5743 -&gt; 10.0.0.10:3307

11.0.0.2:5744 -&gt; 10.0.0.10:3308

11.0.0.3:5742 -&gt; 10.0.0.10:3309

都不需要上送ovs-vswitchd，直接匹配这条megaflow cache就可以完成转发。在这个简单的例子里面，似乎没有问题。

如果增加一条OpenFlow规则：

priority=200,ip,nw\_dst=10.0.0.0/16 actions=output:1

priority=100,ip,nw\_dst=20.0.0.0/16,tcp\_dst=22 actions=drop

相应的，在查找OpenFlow规则时，因为现在有两种Match，会生成两个hash table，因为要确保更高优先级的OpenFlow规则不被遗漏，所以两个hash table都会被查询，也就是说有的OpenFlow规则的match部分都会被用到。对于同样的网络连接：

11.0.0.2:5742 -&gt; 10.0.0.10:3306

上送ovs-vswitchd生成的megaflow cache如下，这里，Megaflow还是会且仅会匹配在经过OpenFlow pipeline时，用到过的field。

ip,nw\_dst=10.0.0.0/16,tcp\_dst=3306 actions=output:1

尽管tcp\_dst并不重要，但是因为查询OpenFlow规则时，它最终影响到了查询结果，所以在Megaflow cache中也会出现。当同样的其他网络连接到来时：

11.0.0.2:5743 -&gt; 10.0.0.10:3307

11.0.0.2:5744 -&gt; 10.0.0.10:3308

11.0.0.3:5742 -&gt; 10.0.0.10:3309

因为匹配不了megaflow cache，都需要上送ovs-vswitchd。这样，仅仅因为一条无关的OpenFlow规则，使得megaflow的优势不复存在了，这明显不合理。因此OpenVSwitch针对megaflow cache的存在，优化了OpenFlow pipeline的查找方式。

Tuple Priority Sorting（优先级排序）

首先是针对OpenFlow的优先级做了优化。按照之前的介绍，如果查找一个OpenFlow Table，总是要查找这个OpenFlow Table对应的所有hash table。即使找到了一个匹配，因为不确定还有没有更高优的匹配，还需要继续查找。针对这点，OpenVSwitch对于一个OpenFlow Table生成的所有hash table，增加了一个属性：pri\_max。它代表当前hash table中所有OpenFlow规则的最高优先级。在查找一个OpenFlow Table的所有hash table时，OpenVSwitch会根据pri\_max从高到低依次遍历。这样，当匹配到某一条OpenFlow规则时，对应的优先级为priority\_F，如果下一个hash table的pri\_max &lt; priority\_F。意味着，接下来的所有hash table中，都不可能找到更高优先级的OpenFlow规则，也就没有必要查找接下来的hash table。

回到上面的例子，如果当前OpenFlow Table有两条规则：

priority=200,ip,nw\_dst=10.0.0.0/16 actions=output:1

priority=100,ip,nw\_dst=20.0.0.0/16,tcp\_dst=22 actions=drop

当下面的网络包上送到ovs-vswitchd时，

11.0.0.2:5742 -&gt; 10.0.0.10:3306

因为在第一个Hash table（pri\_max=200）就可以完成匹配，并且匹配的OpenFlow规则优先级为200，这样没有必要查找第二个Hash table（pri\_max=100）。所以在查找OpenFlow规则时，tcp\_dst并没有被用到，生成的megaflow cache如下：

ip,nw\_dst=10.0.0.0/16 actions=output:1

这样，前面提出的问题被解决了，Megaflow cache的匹配又足够模糊了。为了发挥Tuple Priority Sorting的最大作用：在下发OpenFlow规则时，应当尽量将相同Match的OpenFlow规则配置相同的优先级，并且一个OpenFlow Table中，优先级应当尽量少（OpenFlow定义的优先级是16bit整数）。



**Staged Lookup（分段查找）**

如果将之前的OpenFlow规则稍作修改：

priority=200,ip,nw\_dst=10.0.0.0/16 actions=output:1

priority=300,ip,nw\_dst=20.0.0.0/16,tcp\_dst=22 actions=drop

这个时候，Tuple Priority Sorting的优势就不存在了，因为带有tcp\_dst的OpenFlow规则优先级更高，总是会被先查询到。当下面的网络包上送到ovs-vswitchd时，

11.0.0.2:5742 -&gt; 10.0.0.10:3306

相应的生成的megaflow规则中总是要匹配tcp\_dst。所以在下发OpenFlow规则时，似乎应当将带有tcp\_dst的OpenFlow规则设置成更低优先级，但是这又没有道理。

其实不仅是传输层端口号，为了发挥Tuple Priority Sorting 的优势，匹配了容易变化的field的OpenFlow规则应该低优先级，而匹配了不容易变化的field的OpenFlow规则应该高优先级。这样可以尽量生成只匹配不容易变化的field的megaflow cache，这样的megaflow cache满足尽量的模糊，从而在kernel datapath匹配更多的网络连接。但是这增加了OpenFlow逻辑实现的难度，因为相当于在逻辑实现层面需要考虑底层实现。

另一个办法就是对于具体的OpenFlow规则，先匹配其不容易变化的部分，如果这部分不匹配，就没有必要再去匹配容易变化的部分。但是TSS基于hash table的实现使得区分field变得困难，因为所有相关的field都被hash成一个散列值，没有办法从这个散列值中找出一些field的子集。因此，OpenVSwitch在实现OpenFlow的TSS时，按照field是否容易变化，将原来的一个Hash table分成了四个hash table。第一个hash table只包括metadata，例如ingress port，第二个Hash table包括metadata 和 L2 field，第三个Hash table 包括metadata，L2 field和L3 field，第四个包括metadata，L2 field，L3 field和L4 field，也就是所有的field。最后一个hash table就是之前介绍的Hash table。现在在进行TSS时，原来查找Hash table的部分，会变成顺序的查找这4个Hash table，如果哪个不匹配，就立刻返回，这样可以避免查询不必要的field。这么划分的依据是，在TCP/IP中，外层报文的头部总是比内层报文的头部变化更多。这就是Staged lookup。

所以对应于上面的OpenFlow规则，如果下面的网络包上送到ovs-vswitchd，

11.0.0.2:5742 -&gt; 10.0.0.10:3306

对于priority=300的OpenFlow规则对应的Hash table，因为不匹配metadata，所以没有第一个Hash table。在匹配第三个Hash table时，因为目的IP地址不匹配，可以立即终止，此时只用到了Ethernet Type（IP）和IP header中的目的IP地址的前16bit，tcp\_dst并未用到，所以生成的megaflow cache自然也不会匹配tcp\_dst。Staged Lookup本质是优先匹配更不容易变化的field，可以看做是Tuple Priority Sorting的升级版。

虽然有了Staged Lookup，但是之前查找一个Hash table变成了要查找4个hash table，直观感受是查找性能要下降。但是实际性能差不多，因为大部分的不匹配在查找前两个Hash table就能返回。而前两个hash table的所需要的hash值，计算起来比计算完整的hash值要更节约时间。并且4个Hash table，后面的包含了前面的所有field，而散列值的计算可以是增量计算，就算查找到了第4个Hash table，计算散列值的消耗与拆分前是一样的。

Staged Lookup要发挥作用，在下发OpenFlow规则时，如果匹配包含了tcp\_dst这类易变化的field，应当尽量包含一些metadata，L2 field，或者L3 field。



**Prefix Tracking**

如果现在OpenFlow规则如下：

priority=200,ip,nw\_dst=10.0.0.0/16 actions=output:1

priority=300,ip,nw\_dst=20.0.0.2/32,tcp\_dst=22 actions=drop

那么当下面的网络包上送到ovs-vswitchd时：

11.0.0.2:5742 -&gt; 10.0.0.10:3306

虽然有Tuple Priority Sorting和Staged Lookup，但是因为priority=300的OpenFlow规则匹配的是32位的目的IP地址，在遍历其对应的4个Hash table时，在第3个Hash table不匹配返回，再去查找别的OpenFlow规则对应的Hash table，最终生成的megaflow cache如下所示：

ip,nw\_dst=10.0.0.10/32 actions=output:1

因为在pri\_max=300的Hash Table中，最终用到了目的IP地址的整个32bit。当其他的网络连接到来时：

11.0.0.2:5743 -&gt; 10.0.0.11:3307

11.0.0.2:5744 -&gt; 10.0.0.12:3308

11.0.0.3:5742 -&gt; 10.0.0.13:3309

因为目的IP地址不匹配，都需要上送到ovs-vswitchd重新生成megaflow cache，所以这里也有一些优化空间。OpenVSwitch用了一个Trie 树来管理一个OpenFlow Table中的所有IP地址匹配，实际实现的时候是按照bit生成树。这里为了描述简单，用byte来生成树。之前的OpenFlow对应的Trie树如下所示：

在执行TSS之前，OpenVSwitch会遍历Trie 树来执行LPM（Longest Prefix Match）。通过这个遍历，可以确定（1）生成megaflow cache所需要的IP prefix长度；（2）避免一些不必要的Hash table查找。

回到刚刚那个网络包：

11.0.0.2:5742 -&gt; 10.0.0.10:3306

通过遍历Trie树可以知道，为了匹配目的IP地址（10.0.0.10），可以遍历到10-&gt;0节点，因此只需要匹配前16bit的IP地址就可以在当前OpenFlow Table找到匹配。因此对于这个网络包，生成的megaflow，只需要匹配目的IP地址的前16bit。

另一方面，对于下面的网络包：

11.0.0.2:5742 -&gt; 30.0.0.10:3306

通过遍历Trie树可以知道必然不能匹配目的IP地址，所以也没有必要执行TSS来查找相应的Hash table。

Prefix Tracking是非常有必要的，因为通过OpenFlow实现路由表时，为了实现LPM，通常带有更长的prefix的OpenFlow规则会有更高的优先级，而配合前面的Tuple Priority Sorting，会导致一个OpenFlow Table生成的megaflow cache总是匹配最长的prefix（例如这一小节的例子）。通过Prefix Tracking，可以使得生成的megaflow只匹配足够的IP地址bit数。

不仅对于IP地址，对于传输层端口号，OpenVSwitch也实现了类似的Prefix Tracking，这样当OpenFlow Table中如果出现了只匹配传输层端口号的OpenFlow规则时，生成的megaflow也不用精确匹配传输层端口的16bit，只需要匹配相应的bit位就可以。这样的最终目的还是为了让megaflow足够模糊。

OpenVSwitch还有一些其他的优化手段，但是整体而言，都是为了让生成的megaflow cache尽量模糊，以减少上送ovs-vswitchd的次数，以提升OpenVSwitch的转发性能。

根据OpenVSwitch的转发流程，OpenVSwitch需要维护三个数据：Microflow cache，Megaflow cache和OpenFlow Table。前面介绍的是这三个数据之间的关系以及在转发过程中的作用。这部分来看看OpenVSwitch如何管理这三组数据。



**OpenFlow Table**

从OpenVSwitch的角度来说，OpenFlow的管理是简单的。因为OpenFlow是由SDN controller管理，OpenVSwitch只是提供了OpenFlow增删改查的接口。



**Megaflow Cache**

前面介绍了Megaflow cache的生成过程。但是生成了之后，还面临着更新和删除。对于更新来说，如果SDN controller更新了OpenFlow Table，那么同一个网络包经过OpenFlow pipeline的处理，得出来的datapath actions可能发生变化，对应的megaflow cache也需要更新。理想情况下，OpenVSwitch能够对于特定的事件，精确的确定哪些megaflow需要更新。这在有些场景容易达成，比如说通过匹配MAC地址转发到固定的端口（二层交换机特性），当发现MAC地址迁移到了其他的端口，那么只需要更新匹配了对应MAC地址的megaflow cache就可以。但是由于OpenFlow的通用性，大部分更新的场景不能容易的识别对应的megaflow cache。通常情况下，如果在一个OpenFlow Table中增加了一条OpenFlow规则，任何通过这个OpenFlow Table生成的，且用到了优先级更低的OpenFlow规则的megaflow cache都可能表现出不同的行为，都有可能需要更新。如果考虑到OpenFlow Tables组成的pipeline，这个问题将更加的复杂。

早期的OpenVSwitch将megaflow cache分为两组：第一组cache，当有OpenFlow规则变化时，所有的cache内容都送回到ovs-vswitchd进行重新计算，以确保它们是最新的。第二组cache，通过tag标识变化，将变化与megaflow cache关联起来。例如将MAC地址与magaflow cache联系起来，当MAC发生变化时，通过tag查找到相应的magaflow cache，并更新它。随着SDN应用的越来越复杂，tag也越来越复杂，最终，在OpenVSwitch 2.0，tag标识被废弃，只剩下第一组cache。现在，当有任何变化的时候，整个megaflow cache都会被重新计算。

这看起来是非常低效的，但是好在现在的服务器都是多核，且性能在不断提升。在OpenVSwitch 2.0中，ovs-vswitchd被设计成了多线程，可以充分利用多核CPU。并且多线程的设计可以避免重新计算megaflow cache时，阻塞新的megaflow cache下发。新的megaflow cache下发直接决定了网络连接的性能，所以不能阻塞这个过程。通过把更新megaflow cache和下发新的megaflow cache的程序被放到不同的线程，可以让两个流程谁也不阻塞谁。

默认情况下，重新计算（revalidator）的线程数是：CPU核数/4 + 1。下发新的megaflow cache（handler）的线程数是：CPU核数 - revalidators占用的核数，这样可以充分利用主机的CPU。在一个数千虚拟机规模的环境中，ovs-vswitchd的CPU消耗在10%左右，连一个核的CPU都消耗不了。下面是一个48U主机的OpenVSwitch线程数分配值。

\# ovs-appctl memory/show

handlers:35 ofconns:2 ports:6 revalidators:13 rules:735

通过这样的设计，使得OpenVSwitch可以管理大量的megaflow cache，默认情况下，megaflow cache的数量是20万条。在实际使用中，这个数值是足够使用的。

同时OpenVSwitch还会定期的获取megaflow cache的统计值（大概1秒一次），以获得kernel datapath处理的packets数和bytes数。在OpenFlow规则中看到的packets数和bytes数的统计，就是这个定期同步任务的结果。获取megaflow cache统计值的核心目的是为了确定哪些megaflow cache一段时间没有使用，必须被删除。因为首先megaflow cache的规模是有限的，其次megaflow cache越小，转发性能也越好，所以需要有一个老化的过程。默认情况下，OpenVSwitch设置给megaflow cache的老化时间是10秒。如果megaflow cache已经超过了配置的最大值，这个老化时间也会相应的减少。



**Microflow cache**

对于最新版本的OpenVSwitch，Mircoflow cache只是到megaflow cache中各个Hash table的一个缓存，因此其时效性不是那么的重要，维护起来也更加的简单。

首先，microflow cache有一个固定的大小，如果microflow cache满了就用新的规则随机替换旧的规则，所以，也不需要老化功能；其次，如果因为更新，使得megaflow cache发生变化，microflow cache不会主动的更新，只有当下一个网络包到达microflow cache时，因为对应不到原来的megaflow cache中的Hash table，会重新的在megaflow cache Hash table中查找，进而重新定位到相应的Hash table。因此，microflow的管理也是被动的。

最后，回顾一下OpenVSwitch的实现，它从来不是一个局部最优的解决方案，它的设计哲学是以一个相对较优的方案满足所有的应用场景。

