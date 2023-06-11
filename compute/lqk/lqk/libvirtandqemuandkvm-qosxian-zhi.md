# libvirt配置

libvirt提供了一系列tune的方式，来实现对虚拟机的qos精细控制。下面介绍cpu、内存、磁盘io、网络带宽的qos控制方式。

  


**一. cpu**

| 限制cpu带宽，主要时通过cputune中的quota参数来控制，设置了cpu的quota后就可以限制cpu访问物理CPU的时间片段。libvirt的虚拟机配置如下： |
| :--- |
| &lt;domain type='kvm' id='6'&gt;  ....&lt;cputune&gt;&lt;vcpupin vcpu="0" cpuset="1-4,^2"/&gt;&lt;vcpupin vcpu="1" cpuset="0,1"/&gt;&lt;vcpupin vcpu="2" cpuset="2,3"/&gt;&lt;vcpupin vcpu="3" cpuset="0,4"/&gt;&lt;emulatorpin cpuset="1-3"/&gt;&lt;iothreadpin iothread="1" cpuset="5,6"/&gt;&lt;iothreadpin iothread="2" cpuset="7,8"/&gt;&lt;shares&gt;2048&lt;/shares&gt;&lt;period&gt;1000000&lt;/period&gt;&lt;quota&gt;-1&lt;/quota&gt;&lt;emulator\_period&gt;1000000&lt;/emulator\_period&gt;&lt;emulator\_quota&gt;-1&lt;/emulator\_quota&gt;&lt;iothread\_period&gt;1000000&lt;/iothread\_period&gt;&lt;iothread\_quota&gt;-1&lt;/iothread\_quota&gt;&lt;vcpusched vcpus='0-4,^3' scheduler='fifo' priority='1'/&gt;&lt;iothreadsched iothreads='2' scheduler='batch'/&gt;&lt;/cputune&gt;  ....&lt;/domain&gt; |
| \# virsh schedinfo demo |

  


| 设置cpu亲和性。设置了cpu的亲和性可以使得虚拟机的cpu固定在某些物理cpu上，从而实现对cpu使用的控制和隔离。libvirt虚拟机的配置方式如下： |
| :--- |
| &lt;vcpu placement='static' cpuset='0-1'&gt;2&lt;/vcpu&gt;&lt;cputune&gt;&lt;vcpupin vcpu='0' cpuset='0'/&gt;&lt;vcpupin vcpu='1' cpuset='1'/&gt;&lt;/cputune&gt; |
| 查看信息：\# virsh vcpuinfo instance-0000000d（可查看到CPU Affinity信息） |

  


| 演示虚拟机cpu亲和性绑定 |
| :--- |
| **1.通过下面命令找到虚拟机进程所有的线程\# ps -efL**例如：root     22314     1 22314  0    3 18:21 ?        00:00:00 /usr/libexec/qemu-kvm -name win7 -S -machine pc-i440fx-rhel7.0.0,accel=kvm,usb=off -cpu qemu64,hv\_time,hv\_relaxed,hv\_vapic,hv\_spinlocks=0root     22314     1 22324  0    3 18:21 ?        00:00:02 /usr/libexec/qemu-kvm -name win7 -S -machine pc-i440fx-rhel7.0.0,accel=kvm,usb=off -cpu qemu64,hv\_time,hv\_relaxed,hv\_vapic,hv\_spinlocks=0root     22314     1 22370  0    3 18:21 ?        00:00:00 /usr/libexec/qemu-kvm -name win7 -S -machine pc-i440fx-rhel7.0.0,accel=kvm,usb=off -cpu qemu64,hv\_time,hv\_relaxed,hv\_vapic,hv\_spinlocks=0这里虚拟机win7的线程为22314  22324  22370**2.查看各个线程的绑定情况\# taskset -p 22314**pid 22314's current affinity mask: f\# taskset -p 22324 **pid 22324's current affinity mask: 2**\# taskset -p 22370**pid 22370's current affinity mask: f可以看出22314和22370是没做绑定。22324被绑定到了1号cpu上，说明如下：**22314和22370绑定到了**0123四个cpu上（f即1111，代表绑定到了0123四个cpu上）；22324绑定到了1号cpu上（2即0010，代表绑定到了1号cpu上）。这里主机上一共有0123这4个cpu，所以绑定到0123四个cpu上也就是没有绑定，所以这里22314和22370是没做绑定的。** **3.查看虚拟机运行时的cpu是否真的被绑住了：\# watch -n 1 -d 'ps  -eLo pid,tid,pcpu,psr\| grep 22314'Every 1.0s: ps  -eLo pid,tid,pcpu,psr\| grep 22314                                                                                                                           Wed Nov  9 18:58:17 2016** **22314 22314  0.0   022314 22324  0.2   122314 22370  0.0   1** 这里会看到**22324一致在1上不动，22314和22370会随机的在各个cpu上跑。（左后一列代表在哪个cpu上。虚拟机很忙时效果会很明显）（说明：命令中最后为什么grep 22314，这个22314是刚才ps -eLf的第二列。）** |

**  
**

  


**二. 内存**

| 内存\_qos。设置了内存的qos可以限制虚拟机在物理host山申请内存的大小。libvirt虚拟机的配置方式如下： |
| :--- |
| &lt;domain&gt;  ...&lt;memtune&gt;&lt;hard\_limit unit='G'&gt;1&lt;/hard\_limit&gt;&lt;soft\_limit unit='M'&gt;128&lt;/soft\_limit&gt;&lt;swap\_hard\_limit unit='G'&gt;2&lt;/swap\_hard\_limit&gt;&lt;min\_guarantee unit='byte'&gt;67108864&lt;/min\_guarantee&gt;&lt;/memtune&gt;  ...&lt;/domain&gt; |
| 参数说明：hard\_limit：限制虚拟机在host上使用的最大物理内存。min\_guarantee：最小保证的内存 查看信息：\# virsh memtune instance-00000005 说明：可以通过virsh memtune动态调整上述参数 |

另外还可以通过cgroup来实现对虚拟机的内存限制：

**1.如何通过cgroup做所有虚拟机总内存限制**

\# cat /sys/fs/cgroup/memory/machine/memory.limit\_in\_bytes

9223372036854771712

这里machine是libvrit的默认根cgroup组名。修改/sys/fs/cgroup/memory/machine/memory.limit\_in\_bytes的数值就可以限制所有libvirt创建的虚拟机的使用总内存。

  


**2.如何**

**通过cgroup**

**做部分虚拟机的总内存限制**

| 创建一个名为openstack的自定义cgroup ： |
| :--- |
| \#!/bin/bash cd /sys/fs/cgroupfor i in blkio cpu,cpuacct cpuset devices freezer memory net\_cls perf\_event  do    mkdir $i/machine/openstack.partition  donefor i in cpuset.cpus  cpuset.mems  do    cat cpuset/machine/$i &gt; cpuset/machine/openstack.partition/$i  done |
| 在虚拟机的xml文件中使用： |
| &lt;domain type='kvm' id='6'&gt;  ....&lt;resource&gt;&lt;partition&gt;/machine/openstack.partition&lt;/partition&gt;&lt;/resource&gt;  ....&lt;/domain&gt; |

修改/sys/fs/cgroup/memory/machine/openstack.partition/memory.limit\_in\_bytes的数值，就可以限制通过

openstack.partition

创建的虚拟机的使用总内存

  


3.

**如何**

**通过cgroup**

**做某个虚拟机的内存限制**

（同在虚拟机的xml文件中的memtune中配置hard\_limit）

echo 100000000 

&gt;

 /sys/fs/cgroup/memory/machine/openstack.partition/instance-00000005.libvirt-qemu/memory.limit\_in\_bytes

就可以限制虚拟机instance-00000005的实际使用host的物理内存最大为100M。

  


**三. 磁盘**

| 磁盘\_qos。设置磁盘的qos可以实现对磁盘的读写速率的限制，单位可以时iops或者字节。libvirt虚拟机的配置方式如下： |
| :--- |
| &lt;domain type='kvm' id='6'&gt;  ....&lt;devices&gt;    ....&lt;disk type='network' device='disk'&gt;&lt;driver name='qemu' type='raw' cache='writeback' discard='unmap'/&gt;&lt;source protocol='rbd' name='images/1a956ba7-25fe-49f1-9513-7adb8928036c'&gt;&lt;host name='192.168.107.50' port='6789'/&gt;&lt;host name='192.168.107.51' port='6789'/&gt;&lt;host name='192.168.107.52' port='6789'/&gt;&lt;host name='192.168.107.53' port='6789'/&gt;&lt;/source&gt;&lt;target dev='vda' bus='virtio'/&gt;&lt;iotune&gt;&lt;read\_bytes\_sec&gt;20480&lt;/read\_bytes\_sec&gt;&lt;write\_bytes\_sec&gt;10240&lt;/write\_bytes\_sec&gt;&lt;/iotune&gt;&lt;boot order='1'/&gt;&lt;address type='pci' domain='0x0000' bus='0x00' slot='0x0a' function='0x0'/&gt;&lt;/disk&gt;    ....&lt;/devices&gt;  ....&lt;/domain&gt; |
| &lt;disk type='network' device='disk'&gt;&lt;driver name='qemu' type='raw' cache='writeback' discard='unmap'/&gt;&lt;source protocol='rbd' name='images/1a956ba7-25fe-49f1-9513-7adb8928036c'&gt;&lt;host name='192.168.107.50' port='6789'/&gt;&lt;host name='192.168.107.51' port='6789'/&gt;&lt;host name='192.168.107.52' port='6789'/&gt;&lt;host name='192.168.107.53' port='6789'/&gt;&lt;/source&gt;&lt;target dev='vda' bus='virtio'/&gt;&lt;iotune&gt;&lt;read\_iops\_sec&gt;20480&lt;/read\_iops\_sec&gt;&lt;write\_iops\_sec&gt;10240&lt;/write\_iops\_sec&gt;&lt;/iotune&gt;&lt;boot order='1'/&gt;&lt;address type='pci' domain='0x0000' bus='0x00' slot='0x0a' function='0x0'/&gt;&lt;/disk&gt; |
| \# virsh blkiotune demo |

  


**四. 网卡**

| 网卡\_qos。设置网卡的qos可以限制网卡的io速率。libvirt虚拟机的配置方式如下： |
| :--- |
| &lt;interface type='bridge'&gt;&lt;mac address='fa:16:3e:8f:6a:c9'/&gt;&lt;source bridge='brq5233ef6c-62'/&gt;&lt;bandwidth&gt;&lt;inbound average='2048'/&gt;&lt;outbound average='1024'/&gt;&lt;/bandwidth&gt;&lt;target dev='tap9573c869-24'/&gt;&lt;model type='virtio'/&gt;&lt;address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/&gt;&lt;/interface&gt; |

  


# QOS实现算法

## 令牌桶算法

令牌桶算法是一个非常老牌的 I/O 控制算法，在网络、存储 I/O 上都有着广泛的应用。即：一个固定容量的桶装着一定数量的令牌，桶的容量即令牌数量上限。桶里的令牌每隔固定间隔补充一个，直到桶被装满。一个 IO 请求将消耗一个令牌，如果桶里有令牌，则该 IO 请求消耗令牌后放行，反之则无法放行。对于限制 IO 请求 bps，只需让一个 IO 请求消耗 M 个令牌即可，N 即为此 IO 请求的字节数。

![](/assets/compute-lqk-qos1.png)

令牌桶算法可以达到以下效果：

令牌桶算法可以通过控制令牌补充速率来控制处理 IO 请求的速率；

令牌桶算法允许一定程度的突发，只要桶里的令牌没有耗尽，IO 请求即可立即消耗令牌并放行，这段时间内 IO 请求处理速率将大于令牌补充速率，令牌补充速率实际为平均处理速率；

令牌桶算法无法控制突发速率上限和突发时长，突发时长由实际 IO 请求速率决定，若实际 IO 请求大于令牌补充速率且速率恒定，则：突发时长 = 令牌桶容量 / \(实际 IO 请求速率 - 令牌补充速率\)。

在令牌桶算法的描述中，有一个条件是无强制约束的，那就是在桶里的令牌耗尽时，无法放行的 IO 请求该怎么处理。对于处理网络层报文，实际无外乎三种方式：

第一种是直接丢弃（Traffic Policing，流量监管）。

第二种则是排队等待直到令牌桶完成所缺失数量令牌补充后再消耗令牌放行（traffic shaping，流量整形）。

第三种为前两者的折中，设定有限长度队列，在队列已满时丢弃，否则遵循第二种处理方式。当然对于不可随意丢弃的 IO 请求，处理方式一般为第二种。

此外，一般在实际的实现中，难以做到严格按照固定时间间隔一次补充一个令牌。可考虑的替代方案一般为，加大补充令牌的时间间隔，减少补充次数，但一次补充多个令牌，单次补充令牌的个数与时间间隔成正比。

对于令牌桶算法，还可以考虑一种极端的情况。令牌桶算法支持突发的能力是由令牌桶的容量决定的，以限制 IO 请求 iops 为例，假设我们将令牌桶容量仅设为 1，此时令牌桶算法支持突发的能力也就不复存在了，IO 请求速率上限即被严格控制为令牌补充速率。

## 漏桶算法

漏桶算法有两种类型的定义：leaky bucket as a meter 和 leaky bucket as a queue。

leaky bucket as a meter：同样以限制 IO 请求iops为例，替换上述的网络层报文，我们可以理解如下：一个桶，容量固定。桶以固定的速率漏水，除非桶为空。一个 IO 请求将往桶里增加固定量的水，假如增加的水量将导致桶里的水溢出，则该 IO 请求无法放行，反之则放行。

再把令牌桶的理解放在这里对比下：一个固定容量的桶装着一定数量的令牌，桶的容量即令牌数量上限。桶里的令牌每隔固定间隔补充一个，直到桶被装满。一个 IO 请求将消耗一个令牌，如果桶里有令牌，则该 IO 请求消耗令牌后放行，反之则无法放行。

实际上，把漏水换成补充令牌，把加水换成消耗令牌，再细细品读，我们可以发现，这两段描述的含义就是一样的，仅仅是角度不一样而已。所以 leaky bucket as a meter 与 token bucket 可认为是等价的。

leaky bucket as a queue：漏桶实际为一个有限长队列。当一个报文到达时，假如队列未满，则入队等待，反之则被丢弃。在队列不为空时，每个固定时间间隔处理一个报文。

![](/assets/compute-lqk-qos2.png)虽然描述更为简单，但可以看到 leaky bucket as a queue 比 leaky bucket as a meter 的要求要严格得多，leaky bucket as a queue 可以达到以下效果：

可以通过控制处理时间间隔来严格控制速率上限；

不支持任何程度的突发；

对超出限定速率的报文可能做入队等待处理，也可能做直接丢弃处理，当队列长度设置为足够小，设置为 0 时，可认为等同于全部做直接丢弃处理（Traffic Policing），而队列长度设置为足够大时，可认为等同于全部做入队等待处理（Traffic Shaping）。

如此一来，我们可以看到 leaky bucket as a queue 实际效果与令牌桶容量为 1 的令牌桶算法是一致的，也就是说 leaky bucket as a queue 其实可视为令牌桶算法的一种特例（不支持突发）。

## 前端 QoS：通过 QEMU 的块设备 IO 限速机制进行限速

QEMU 早在 1.1 版本就已支持块设备的 IO 限速，提供 6 个配置项，可对上述 6 种场景分别进行速率上限设置。在 1.7 版本对块设备 IO 限速增加了支持突发的功能，以总 iops 场景为例：支持设置可突发的总 iops 数量。在 2.6 版本对支持突发的功能进行了完善，可控制突发速率和时长。

![](/assets/compute-lqk-qos3.png)QEMU 的块设备 IO 限速机制主要是通过漏桶算法实现。在 2.6 版本中不仅支持突发，可支持控制突发速率和时长。

![](/assets/compute-lqk-qos5.png)

QEMU 同样使用了多个桶，以对 6 种场景独立进行 QoS 限速。

QEMU 通过大桶和突发小桶的设计，支持了对突发速率和突发时长的控制。有突发流量到来时，限速分为 3 个阶段：

首先是突发小桶未满时，未做速率限制，但此突发小桶的容量仅设置为突发速率值的 1/10，所以此阶段所经历的时间特别短，效果可忽略不计；

其次是突发小桶已满而大桶未满的阶段，速率即被限制为突发速率，由于大桶在不断地以基本速率漏水，所以实际的突发时长要大于设置的突发时长，此阶段的实际突发时长为：实际突发时长 = 大桶容量 / \(突发速率 - 基本速率上限\)，而 大桶容量 = 突发速率 \* 设置突发时长。

最后是大桶已满的阶段，此时速率就被限制为基本速率上限了。

在不考虑突发的情况下，从算法效果来看，QEMU 实现的漏桶算法实际为桶容量很小的令牌桶算法，当然也因为桶容量足够小，所以基本可视为我们狭义理解上的漏桶算法。

与 Librbd 固定时间间隔补充令牌不同，QEMU 在处理每个请求时同步执行漏水操作，每秒执行漏水操作的次数实际与每秒 IO 请求数量相等，漏水频率一般远大于 Librbd 补充令牌的频率。这也形成了一个特点，在 QoS 限速情况下，QEMU 处理 IO 请求的时间点分布比较均匀，而 Librbd 则相对集中在补充令牌的时间点上。同时 IO 请求延迟的分布特点也与 Librbd 有明显的不同。

## 后端 QoS：通过 librbd 的镜像 IO 限速机制进行限速

Ceph 在 13.2.0 版本（M 版）支持对 RBD 镜像的 IO 限速，此版本仅支持总 iops 场景的限速，且支持突发，支持配置突发速率，但不可控制突发时长（实际相当于突发时长设置为 1 秒且无法修改）。在 14.2.0 版本（N 版）增加了对读 iops、写 iops、总 bps、读 bps、写 bps 这 5 种场景的限速支持，对突发的支持效果保持不变。

![](/assets/compute-lqk-qos6.png)Ceph Librbd 的镜像 IO 限速机制使用的是令牌桶算法。所以，Librbd 的限速机制支持突发，支持配置突发速率，但不支持控制突发时长，这与令牌桶算法定义描述的效果近乎一致。而从实际的实现来看，Librbd 的令牌桶算法实现也的确与定义比较匹配。

![](/assets/compute-lqk-qos7.png)

Librbd 使用了 6 个令牌桶，所以可对 6 种场景独立进行 QoS 限速，若这些场景的 QoS 限速全部配置启用，那么一个 I/O 请求需在每一个令牌桶消耗相应数目的令牌后才可被放行（当然，写请求在限制读速率的桶无需消耗令牌，反之亦然）。

令牌桶的容量由基本速率上限值或突发速率值决定，而且突发速率值的唯一作用就是用于配置令牌桶容量。从上面的内容可知，令牌桶算法的效果分析来看，只要令牌桶容量不是足够小，该令牌桶算法即支持突发。所以即使未设置突发速率，令牌桶容量由于为基本速率值，实际也是支持突发的：突发时长 = 基本速率值 / \(实际 IO 请求速率 - 基本速率\)。

此外，令牌桶算法本身无法控制突发速率上限，所以设置所谓的突发速率其实并没有对真正对速率上限进行限制，只是通过增大令牌桶容量而客观地增大了突发时长：突发时长 = 突发速率值 / \(实际 IO 请求速率 - 基本速率\)。

此处以限制 iops 来举个实际例子，基本速率设置为 1000iops，突发速率设置为 2000iops，假如实际 IO 请求速率恰好为 2000iops，那么突发时长实际就支持 2 秒。而且实际 IO 请求速率并没有得到限制，假如达到了 5000iops，那么实际突发时长仅为 0.5 秒，即 0.5 秒后，速率就被稳定限制在 1000iops。可见 Librbd 的突发速率概念还是很容易给人带来误解的，实际应理解为可突发的 I/O 请求量。

Librbd 的令牌桶算法实现中，加大了补充令牌的时间间隔，一次补充多个令牌，此方式有助于减小算法的运行开销。在令牌补充的一瞬间到令牌消耗完这段时间内，IO 请求由于获得了足够的令牌数量，可被快速放行，而一旦令牌耗尽，超过 QoS 限速的请求则陷入排队等待的状态，等待时间即为令牌补充的时间间隔，甚至更多。

总结

Librbd 的限速算法与 QEMU 相比有两大不同，一是 Librbd 无法像 QEMU 一样控制突发速率上限和突发时长，二是 Librbd 补充令牌的方式和 QEMU 漏水的方式不同，前者导致请求被处理的时间点较为集中，类似于一批一批地处理，后者则使请求被处理的时间点分布均匀，类似于一个一个地处理。

从测试结果上可以看到，同等条件下，IO 请求平均时延接近于相等，而 QEMU 的限速机制让每个 IO 请求的延迟都接近于平均时延，Librbd 的限速机制则使大部分 IO 请求有较低的延迟，但有一定比例的 IO 请求延迟很高，接近于补充令牌的时间间隔。

