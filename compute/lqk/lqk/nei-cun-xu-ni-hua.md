# [地址转换](https://so.csdn.net/so/search?q=地址转换&spm=1001.2101.3001.7020)

在Linux中的地址转换通常是Virtual Address（虚拟地址）通过MMU和页表转换得到Physical Address（物理地址）。

![](/assets/compute-libvirtqemukvm-memoryconvert1.png)

之所以不直接用物理地址是因为：

* 多个进程同时运行时，他们的映像文件地址可能会一致，发生冲突。

* 直接使用物理地址，不便于对进程地址空间进行隔离

* 物理内存有限，使用虚拟地址，在内存紧张时可以通过分时方式让多个进程共享页面

### MMU

MMU是处理器/核（processer）中的一个硬件单元，通常每个核有一个MMU。MMU由两部分组成：TLB\(Translation Lookaside Buffer\)和table walk unit。

TLB快表，直译为旁路快表缓冲，也可以理解为页表缓冲，地址变换高速缓存。

由于页表存放在主存中，因此程序每次访存至少需要两次：一次访存获取物理地址，第二次访存才获得数据。提高访存性能的关键在于依靠页表的访问局部性。当一个转换的虚拟页号被使用时，它可能在不久的将来再次被使用到，。

TLB是一种高速缓存，内存管理硬件使用它来改善虚拟地址到物理地址的转换速度。当前所有的个人桌面，笔记本和服务器处理器都使用TLB来进行虚拟地址到物理地址的映射。使用TLB内核可以快速的找到虚拟地址指向物理地址，而不需要请求RAM内存获取虚拟地址到物理地址的映射关系。

table walk

1. 从协处理器CP15的寄存器2\(TTB寄存器，translation table base register  **ARM架构**，**X86中是CR3**\)中取出保存在其中的第一级页表\(translation table\)的基地址。这个基地址指的是PA，也就是说页表是直接按照这个地址保存在物理内存中的。

2. 以TTB中的内容为基地址，以VA\[31:20\]为索引值在一级页表中查找对应表项。这个页表项保存着第二级页表\(coarse page table\)的基地址，这同样是物理地址，也就是说第二级页表也是直接按这个地址存储在物理内存中的。

3. 以VA\[19:12\]为索引值在第二级页表中查出表项，这个表项中就保存着物理页面的基地址，我们知道[虚拟内存](https://so.csdn.net/so/search?q=虚拟内存&spm=1001.2101.3001.7020)管理是以页为单位的，一个虚拟内存的页映射到一个物理内存的页框，从这里就可以得到印证，因为查表是以页为单位来查的。

4. 有了物理页面的基地址之后，加上VA\[11:0\]这个偏移量就可以取出相应地址上的数据了。

   ![](/assets/compute-libvirtkvmqemu-memconvert2.png)

这个过程称为Translation Table Walk，Walk这个词用得非常形象。从TTB走到一级页表，又走到二级页表，又走到物理页面，一次寻址其实是三次访问物理内存。注意这个“走”的过程完全是硬件做的，每次CPU寻址时MMU就自动完成以上四步，不需要编写指令指示MMU去做，前提是操作系统要维护页表项的正确性，每次分配内存时填写相应的页表项，每次释放内存时清除相应的页表项，在必要的时候分配或释放整个页表。

### 页表

page table是每个进程独有的，是软件实现的，是存储在main memory（比如DDR）中的。

因为访问内存中的页表相对耗时，尤其是在现在普遍使用多级页表的情况下，需要多次的内存访问，为了加快访问速度，系统设计人员为page table设计了一个硬件缓存 -**TLB**，CPU会首先在TLB中查找，因为在TLB中找起来很快。TLB之所以快，一是因为它含有的entries的数目较少，二是TLB是集成进CPU的，它几乎可以按照CPU的速度运行。

如果在TLB中找到了含有该虚拟地址的entry（TLB hit），则可从该entry中直接获取对应的物理地址，否则就不幸地TLB miss了，就得去查找当前进程的page table。这个时候，组成MMU的另一个部分table walk unit就被召唤出来了，这里面的table就是page table。

使用table walk unit硬件单元来查找page table的方式被称为hardware TLB miss handling，通常被CISC架构的处理器（比如IA-32）所采用。它要在page table中查找不到，出现page fault的时候才会交由软件（操作系统）处理。

与之相对的通常被RISC架构的处理器（比如Alpha）采用的software TLB miss handling，TLB miss后CPU就不再参与了，由操作系统通过软件的方式来查找page table。使用硬件的方式更快，而使用软件的方式灵活性更强。IA-64提供了一种混合模式，可以兼顾两者的优点。

## [虚拟机](https://so.csdn.net/so/search?q=虚拟机&spm=1001.2101.3001.7020)地址转换

如果这个操作系统是运行在虚拟机上的，那么这只是一个中间的物理地址（Intermediate Phyical Address - IPA），需要经过VMM/hypervisor的转换，才能得到最终的物理地址（Host Phyical Address -**HPA**）。从VMM的角度，guest VM中的虚拟地址就成了**GVA**\(Guest Virtual Address\)，IPA就成了**GPA**\(Guest Phyical Address\)。

![](/assets/compute-libvirtkvmqemu-memconvert3.png)![](/assets/compute-lqk-memory21.png)

可见，如果使用VMM，并且guest VM中的程序使用虚拟地址（如果guest VM中运行的是不支持虚拟地址的RTOS，则在虚拟机层面不需要地址转换），那么就需要两次地址转换。

![](/assets/compute-libvirtkvmqemu-memconvert5.png)

但是传统的IA32（x86\_32）架构从硬件上只支持一次地址转换，即由CR3寄存器指向进程第一级页表的首地址，通过MMU查询进程的各级页表，获得物理地址。

**Part III：页表**

每一个进程都有一份页表（Page Table），作为其上下文的一部分。页表由一系列页表条目（PTE，Page Table Entry）组成，每个页表条目都包含着虚拟页和物理页的映射关系。PTE 由一个有效位（valid bit，表明该虚拟页是否被缓存在 DRAM 中）和一个 n 位地址字段组成。

页表（page table）的数据结构存放在物理内存（DRAM）中，操作系统负责维护页表结构，每次 MMU（内存管理单元）中的地址翻译硬件将一个虚拟地址转换成物理地址时，都会读取页表。

MMU 通过页表来确定一个虚拟页是否缓存在 DRAM 中：

* 如果是（有效位为 1）
  ，则该条目指向该虚拟页所存放在物理页的位置；
* 如果不是（有效位为 0）
  ，则该条目指向该虚拟页所存放在磁盘的位置，在物理内存（DRAM）中选择一个牺牲页，并将虚拟页从磁盘复制到 DRAM 中，替换这个牺牲页。
 

![](/assets/compute-arch-vmm-pagetable1.png)

上图为包含 8个虚拟页，4 个物理页的系统的页表结构。

  


**Part IV：页命中**

页命中指的是当 MMU 需要根据虚拟地址输出物理地址时，这个地址所在的页已经被装载到物理内存中了（即对应的 PTE 的有效为 1）。

![](/assets/compute-arch-vmm-pagehit1.png)

上图中，当 MMU 访问的虚拟地址对应到页表中 VP 2 时，地址翻译硬件发现该地址在页表当中有效位为 1，即被缓存在 DRAM 当中（称为页命中），则使用页表当中 PTE 所对应的物理内存地址，来访问数据。

  


**Part V：缺页**

![](/assets/compute-arch-vmm-pagefault1.png)

上图中，当 MMU 访问的虚拟地址对应到页表中 VP 3 时，地址翻译硬件发现该地址在页表当中有效位为 0，即未被缓存在 DRAM 当中，称为缺页（Page Fault），触发一个缺页异常。

缺页异常的处理程序被启动，该程序会选择一个牺牲页，若是该牺牲页被标记为已经更改过，则内核会将其复制回磁盘，若是未更改过，调整牺牲页在页表中所对应的 PTE。接着，内核从磁盘\(虚拟内存\)当中将内容复制到牺牲页\(物理内存\)上，再次更新其PTE，随后返回。

当缺页异常处理程序返回时，原进程会重新启动导致缺页异常的指令，该指令会将导致缺页的虚拟地址重发送到地址翻译硬件，这时就会进行页命中的相关流程了。

![](/assets/compute-arch-vmm-pagefault2.png)

上图中，触发缺页异常后，缺页异常处理程序选择 VP 4 作为牺牲页，并从磁盘上用 VP 3 的副本取代它。

![](/assets/compute-arch-vmm-pagefault3.png)

在缺页异常处理程序重新启动导致缺页的指令之后，该指令将从内存中正常地读取字，而不会再产生缺页异常。

  


**Part VI：分配新页**

![](/assets/comput-arch-vmm-newpage.png)

如上图，内核在磁盘上分配 VP 5，并将 PTE 5 指向这个新的位置。

  


**Part VII：虚拟内存作为内存管理的工具**

操作系统为每个进程提供了一个独立的页表，也就是每个进程独占一个独立的虚拟地址空间。

![](/assets/compute-arch-vmm-processvm1.png)

这样做的好处：

这样做的好处：

* 简化共享内存
  ：操作系统通过将不同进程中适当的虚拟页面映射到相同的物理页面，从而安排多个进程共享下共同代码的副本。
* 简化链接
  ：独立的地址空间允许每个进程的内存布局使用相同的基本格式，而不管代码和数据实际存放在物理内存的何处。链接器可以假设每个程序都加载到相同的位置，然后它可以重定位这些引用。
* 简化加载
  ：execve 查看 ELF 文件，它知道文件中的代码和数据段有多大，它从固定的地址为代码和数据分配虚拟内存。
* 简化内存分配
  ：虚拟内存为向用户进程提供一个简单的分配额外内存的机制。当一个运行在用户进程中的程序要求额外的堆空间时（如调用malloc），操作系统分配一个适当数字（eg：k）个连续的虚拟内存页面，并且将他们映射到物理内存中任意位置的 k 个任意的物理页面。



**Part II：使用页表的地址翻译**

在 CPU 中地址翻译由一个叫做 MMU（Memory Management Unit，内存管理单元）的硬件完成。MMU 接收一个虚拟地址，并且输出一个物理地址。如果这个虚拟地址在物理内存中存在，那么就叫做页命中。如果这个虚拟地址在物理内存中不存在，那么 MMU 将产生一个缺页错误。

![](/assets/compute-archtiecture-mmu1.png)n 位的虚拟地址包括两个部分：一个 p 位的虚拟页面偏移（Virtual Page Offset，VPO），和一个 n-p 位的虚拟页号（Virtual Page Number， VPN），MMU 利用 VPN 来选择适当的 PTE。

因为物理和虚拟页面都是 P 字节的，所以物理页面偏移（Physical Page Offset，PPO）和 虚拟页面偏移（Virtual Page Offset，VPO）是相同的，因此将 PTE 中的物理页号（Physical Page Number， PPN）与 VPO 串联起来，就得到了相应的物理地址。

#### 

#### **页命中时地址翻译**

![](/assets/comupte-architecture-mmu2.png)

1. 处理器生成一个虚拟地址，并把它传送给 MMU。

2. MMU 生成根据虚拟地址生成 VPN，然后请求高速缓存/主存，获取 PTE 的数据。

3. 高速缓存/主存向 MMU 返回 PTE 的数据。

4. 从 PTE 获取对应的物理页号 PPN。用物理页的基址加上页偏移 PPO（假设页大小为 4KB，那么页偏移就是虚拟地址的低 12 位，物理页的页偏移和虚拟页的页偏移相同），获取对应的物理地址。

5. 主存/高速缓存将数据返回给 CPU。

**2.缺页时的地址翻译**  


![](/assets/compute-architecture-mmu3.png)

1. 处理器生成一个虚拟地址，并把它传送给 MMU。

2. MMU 根据虚拟地址生成 VPN，然后请求高速缓存/主存，获取 PTE 的数据。

3. 高速缓存/主存向 MMU 返回 PTE 的数据。

4. 由于判断出 PTE 的有效位是 0，所以 CPU 将触发一次异常，将控制权转移给内核中的缺页异常处理程序。

5. 缺页异常处理程序确定出物理内存中的牺牲页，如果这个页面被修改过了（D 标志位为 1），那么将牺牲页换出到磁盘。

6. 缺页处理程序从磁盘中调入新的页面到主存中，并且更新 PTE。

7. 缺页处理程序将控制权返回给原来的进程，再次执行导致缺页的指令。再次执行后，就会产生页命中时的情况了。



## 软件实现-影子页表

为了支持**GVA-&gt;GPA-&gt;HPA**的两次转换，可以计算出GVA-&gt;HPA的映射关系，将其写入一个单独的影子页表（**sPT**- shadow Page Table）。在一个运行Linux的guest VM中，每个进程有一个由内核维护的页表，用于GVA-&gt;GPA的转换，这里我们把它称作**gPT**\(guest Page Table\)。

VMM层的软件会将gPT本身使用的物理页面设为write protected的，那么每当gPT有变动的时候（比如添加或删除了一个页表项），就会产生被VMM截获的page fault异常，之后VMM需要重新计算GVA-&gt;HPA的映射，更改sPT中对应的页表项。可见，这种纯软件的方法虽然能够解决问题，但是其存在两个缺点：

* 实现较为复杂，需要为每个guest VM中的每个进程的gPT都维护一个对应的sPT，增加了内存的开销。

* VMM使用的截获方法增多了page fault和trap/vm-exit的数量，加重了CPU的负担。

在一些场景下，这种影子页表机制造成的开销可以占到整个VMM软件负载的75%。

![](/assets/compute-libvirtqemukvm-memconvert7.png)

## 硬件实现-EPT/NPT

为此，各大CPU厂商相继推出了硬件辅助的内存[虚拟化技术](https://so.csdn.net/so/search?q=虚拟化技术&spm=1001.2101.3001.7020)，比如Intel的EPT\(Extended Page Table\)和AMD的NPT\(Nested Page Table），它们都能够从硬件上同时支持GVA-&gt;GPA和GPA-&gt;HPA的地址转换的技术。

GVA-&gt;GPA的转换依然是通过查找gPT页表完成的，而GPA-&gt;HPA的转换则通过查找nPT页表来实现，每个guest VM有一个由VMM维护的nPT。其实，EPT/NPT就是一种扩展的MMU（以下称EPT/NPT MMU），它可以交叉地查找gPT和nPT两个页表：

假设gPT和nPT都是4级页表，那么EPT/NPT MMU完成一次地址转换的过程是这样的（不考虑TLB）：

首先说明gCR3和nCR3，他们分别是客户机和主机CR3的拷贝。

首先它会查找guest VM中CR3寄存器（gCR3）指向的PML4页表，由于gCR3中存储的地址是GPA，因此CPU需要查找nPT来获取gCR3的GPA对应的HPA。nPT的查找和前面的页表查找方法是一样的，这里我们称一次nPT的查找过程为一次**nested walk**。

![](/assets/compute-libvirtkvmqemu-memconvet8.png)

如果在nPT中没有找到，则产生EPT violation异常（可理解为VMM层的page fault）。如果找到了，也就是获得了PML4页表的物理地址后，就可以用GVA中的bit位子集作为PML4页表的索引，得到PDPE页表的GPA。接下来又是通过一次nested walk进行PDPE页表的GPA-&gt;HPA转换，然后重复上述过程，依次查找PD和PE页表，最终获得该GVA对应的HPA。

![](/assets/compute-libvritkvmqem-memconvert9.png)

不同于影子页表是一个进程需要一个sPT，EPT/NPT MMU解耦了GVA-&gt;GPA转换和GPA-&gt;HPA转换之间的依赖关系，一个VM只需要一个nPT，减少了内存开销。如果guest VM中发生了page fault，可直接由guest OS处理，不会产生vm-exit，减少了CPU的开销。可以说，EPT/NPT MMU这种硬件辅助的内存虚拟化技术解决了纯软件实现存在的两个问题。

针对Nested walk总结一下：

\(1\) Nested walk 的目的：GPA-&gt;HPA转换

\(2\) 页表/页目录/页中间目录表的及地址都是"物理地址"，所以，gCR3和nCR3中存放"物理地址"，gCR3存放guest OS认为的物理地址，nCR3存放host OS认为的物理地址\(实际机器的物理地址\)

## EPT/NPT MMU优化

事实上，EPT/NPT MMU作为传统MMU的扩展，自然也是有TLB的，它在查找gPT和nPT之前，会先去查找自己的**TLB**（前面为了描述的方便省略了这一步）。但这里的TLB存储的并不是一个GVA-&gt;GPA的映射关系，也不是一个GPA-&gt;HPA的映射关系，而是最终的转换结果，也就是GVA-&gt;HPA的映射。

不同的进程可能会有相同的虚拟地址，为了避免进程切换的时候flush所有的TLB，可通过给TLB entry加上一个标识进程的PCID\(x86\)/ASID\(ARM\)的tag来区分。同样地，不同的guest VM也会有相同的GVA，为了flush的时候有所区分，需要再加上一个标识虚拟机的tag，这个tag在ARM体系中被叫做VMID，在Intel体系中则被叫做VPID。

![](/assets/compute-libvirtkvmqemu-memconvert10.png)

在最坏的情况下（也就是TLB完全没有命中），gPT中的每一级转换都需要一次nested walk，而每次nested walk需要4次内存访问，因此5次nested walk总共需要（4+1）\*5-1=24次内存访问（就像一个5x5的二维矩阵一样）：

虽然这24次内存访问都是由硬件自动完成的，不需要软件的参与，但是内存访问的速度毕竟不能与CPU的运行速度同日而语，而且内存访问还涉及到对总线的争夺，次数自然是越少越好。

要想减少内存访问次数，要么是增大EPT/NPT TLB的容量，增加TLB的命中率，要么是减少gPT和nPT的级数。gPT是为guest VM中的进程服务的，通常采用4KB粒度的页，那么在64位系统下使用4级页表是非常合适的。

而nPT是为guset VM服务的，对于划分给一个VM的内存，粒度不用太小。64位的x86\_64支持2MB和1GB的large page，假设创建一个VM的时候申请的是2G物理内存，那么只需要给这个VM分配2个1G的large pages就可以了（这2个large pages不用相邻，但large page内部的物理内存是连续的），这样nPT只需要2级（nPML4和nPDPE）。

如果现在物理内存中确实找不到2个连续的1G内存区域，那么就退而求其次，使用2MB的large page，这样nPT就是3级（nPML4, nPDPE和nPD）。

## 总结

不管是影子页表还是EPT/NPT的优化都能看得出，[虚拟化](https://so.csdn.net/so/search?q=虚拟化&spm=1001.2101.3001.7020)中的地址转换所依托的还是虚拟内存中的地址转换方式。相比于影子页表硬件层面的EPT/NPT技术显然更胜一筹，速度更快，开销更少，由于虚拟化本身的属性，它所依托的还是真实的物理机，所以要理解这一部分还是需要先了解OS内存管理中的地址转换是如何进行的。

## 参考资料

[https://zhuanlan.zhihu.com/p/69828213](https://zhuanlan.zhihu.com/p/69828213)

[https://zhuanlan.zhihu.com/p/66971714](https://zhuanlan.zhihu.com/p/66971714)

[https://www.vmware.com/pdf/Perf\_ESX\_Intel-EPT-eval.pdf](https://www.vmware.com/pdf/Perf_ESX_Intel-EPT-eval.pdf)

[http://developer.amd.com/wordpress/media/2012/10/NPT-WP-1 1-final-TM.pdf](http://developer.amd.com/wordpress/media/2012/10/NPT-WP-1 1-final-TM.pdf)

