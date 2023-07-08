## 计算机系统分类

Michael J. Flynn提出按指令流和数据流的多倍性对计算系统分类（通常称为Flynn分类法）。

指令流：机器执行的指令序列。

数据流：由指令流调用的数据序列，包括输入数据和中间结果。

多倍性：在系统性能瓶颈部件上处于同一执行阶段的指令或数据的最大可能个数。

Flynn分类法

|  | 单指令流 | 多指令流 |
| :--- | :--- | :--- |
| 单数据流 | 单指令单数据流（SISD） | 多指令流单数据流（MISD） |
| 多数据流 | 单指令流多数据流（SIMD） | 多指令流多数据流（MIMD） |

（1）单指令单数据流（Single Instruction Stream Single Data Stream，SISD）

SISD其实就是传统的顺序执行的单处理器计算机，其指令部件每次只对一条指令进行译码，并只对一个操作部件分配数据。流水线方式的单处理机有时也被当作SISD。

SISD机器是一种传统的串行计算机，它的硬件不支持任何形式的并行计算，所有的指令都是串行执行。并且在某个时钟周期内，CPU只能处理一个数据流。因此这种机器被称作[单指令流单数据流](https://baike.baidu.com/item/单指令流单数据流/12028690?fromModule=lemma_inlink)机器。早期的计算机都是SISD机器，如冯诺.依曼架构，如IBM PC机，早期的巨型机和许多8位的家用机等。

[单指令流多数据流](https://baike.baidu.com/item/单指令流多数据流/257612?fromModule=lemma_inlink)（Single Instruction Stream Multiple Data Stream，SIMD）

SIMD是采用一个指令流处理多个数据流。

SIMD以并行处理机（[阵列处理机](https://baike.baidu.com/item/阵列处理机/6892866?fromModule=lemma_inlink)）为代表，并行处理机包括多个重复的处理单元PU1~PUn，由单一指令部件控制，按照同一指令流的要求为它们分配各自所需的不同数据。这类机器在数字信号处理、图像处理、以及多媒体信息处理等领域非常有效。

Intel处理器实现的MMXTM、SSE\(Streaming SIMD Extensions\)、SSE2及SSE3[扩展指令集](https://baike.baidu.com/item/扩展指令集/2313347?fromModule=lemma_inlink)，都能在单个时钟周期内处理多个数据单元。也就是说我们用的单核计算机基本上都属于SIMD机器。

（3）多指令流单数据流（Multiple Instruction Stream Single Data Stream，MISD）

MISD具有n个处理单元，按n条不同指令的要求对同一数据流及其中间结果进行不同的处理。一个处理单元的输出又将作为另一个处理单元的输入。

MISD是采用多个指令流来处理单个数据流。由于实际情况中，采用多指令流处理多数据流才是更有效的方法，因此MISD只是作为理论模型出现，没有投入到实际应用之中。

（4）多指令流多数据流（Multiple Instruction Stream MultipleData Stream，MIMD）

MIMD系统是指能实现作业、任务、指令、数组各级全面并行的多机系统。

MIMD机器可以同时执行多个指令流，这些指令流分别对不同数据流进行操作。最新的多核计算平台就属于MIMD的范畴，例如Intel和AMD的双核处理器等都属于MIMD。

并行计算机系统绝大部分为MIMD系统，包括并行向量处理机（PVP，Parallel Vector Processor），对称对多处理机（SMP，Symmetrical Multi Processor），规模并行处理机（MPP,Massively Parallel Processor），工作站机群（COW，Cluster Of Workstations）,分布式共享存储系统（DSM,Distributed shared Memory）。





