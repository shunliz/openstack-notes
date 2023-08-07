![](/assets/compute-hardware-apic1.png)

APIC全称是Advanced Programmable Interrupt Controller，高级可编程中断控制器。它是在奔腾P54C之后被引入进来的。在现在的计算机它通常由两个部分组成，分别为LAPIC（Local APIC，本地高级可编程中断控制器）和IOAPIC\(I/O高级可编程中断控制器）。LAPIC在CPU中，IOAPIC通常位于南桥。APIC是在PIC \(Programmable Interrupt Controller\) 的基础上发展而来的。在传统的单处理器的PC-AT 兼容的机器，通常使用8259A双片级联组成Legacy中断控制器（如下图所示）。一共可以连接15个设备（有一个管脚被用于串联另一片8259\)。PIC的优先级规则比较简单，0号管脚的优先级最高。PIC可以通过ICW（Initialization CommandWord）编程设置起始的Vector号。

![](/assets/compute-hardware-apic2.png)

在该模式下,当8259检测到中断来自设备的中断请求，它会拉高INTR（Interrupt Request）通常CPU,作为响应，CPU会采取下述的行动：

1. 在中断是使能的情况下，当CPU执行完当前的指令后，它会识别出该中断请求。
2. CPU暂时停止执行被中断的程序。
3. CPU产生一个INTA（Interrupt Acknowledge ）给中断控制器去获得有最高优先级请求的中断向量。
4. 中断控制器提供带有最高优先级请求的8位的中断向量给CPU。
5. CPU使用这个中断向量作为IDT的索引，找到相应的CS:EIP也就是对应设备中断处理程序的入口。
6. CPU 把 CS,EIP 和 EFLAGS 等寄存器压入栈中保存。
7. CPU自动disable识别其他的外部硬件中断。
8. CPU开始执行中断处理程序。

完成以后, CPU写EOI（End Of Interrupt\)。收到EOI后，ISR中最高优先级的bit自动清零。

PIC在单个CPU的环境下工作的挺好，可是当系统越来越复杂出现了多个CPU的情况，它就没法很好的工作了。这种情况下所有的中断都会丢给主8259连接的那个CPU，那么这个CPU就要处理所有的中断，性能上就会有影响。在一个多处理器的系统，任何一个CPU都应该能够响应来自任何设备的中断，这种就是所谓的对称式多重处理（Symmetric Multiprocessing）

![](/assets/compute-hardware-apic3.png)

**IOAPIC:**

IOAPIC的主要作用是中断的分发。最初有一条专门的APIC总线用于IOAPIC和LAPIC通信，在Pentium4 和Xeon 系列CPU出现后，他们的通信被合并到系统总线中。

![](/assets/compute-hardware-apic4.png)

PRT table（Programmable Redirection Table）: IOAPIC的内部有一个PRT table（Programmable Redirection Table），IOAPIC可以根据table的格式，发送中断消息给特定的CPU的LAPIC，有LAPIC再通知CPU进行处理。PRT 格式如下图所示， 通常一个IOAPIC有24个中断管脚，每个管脚对应一个RTE \(Redirection Table Entry\)。这些管脚本身并没有优先级的区分，但是RTE中会有Vector，LAPIC会基于Vector设定优先级。



Virtual Wire Mode:虚拟接线模式，该模式主要是为了向前兼容，可以理解为就是PIC。在这种模式下，IOAPIC会把8259A的模拟硬件产生的中断信号直接送给BSP。



![](/assets/compute-hardware-apic5.png)



**Local APIC:**

如上图所示，每个CPU/Core中都有一个LAPIC。相比较于IOAPIC， LAPIC会更复杂一些，它不仅能够处理来自IOAPIC的中断消息，它还可以处理IPI（Inter-Processor Interrupt Messages）,NMI,SMI和Init消息。 LAPIC 默认使用基地址FEE00000h开始的4KB空间用来访问内部寄存器，从Pentium Pro开始引入一个APIC\_BASE 的MSR，可以用来设置基地址在任何4KB对齐的64GB内存地址空间。 IRR：LAPIC已经收到中断但是还未提交CPU处理。 ISR：CPU已经开始处理中断，但是尚未完成。 TMR\(Trigger Mode Register\)：中断的触发模式。EOI软件写入表示中断处理完成。 完整的寄存器列表如下图所示：



APIC ID是用来标识LAPIC的，在系统reset的时候microde会给每个LAPIC分配一个唯一的APIC ID, 系统软件通常会使用该ID唯一标识一个CPU，最初只有4位最多只能表示15个CPU，后来扩展到了8位最多支持255个LAPIC（后来的X2APIC支持更多65535个）。

外部设备接在IOAPIC上，它的中断是通过PRTE里面的Destination Field指定的APICID决定送给哪个LAPIC 并最终被CPU处理的。根据Destination Mode的不同，它 又分为Physical模式和Logical模式。Physical模式比较简单就直接看APICID，Logical模式可能还要看Cluster ID 这个比较复杂，这里就不讨论了。

每个中断都有一个vector, X86平台默认有256个vector其中前面32是被预留或者被异常占用的。中断的优先级别是 vector/16，所以中断一共拥有2~15个级别。对于8bit的vector，又可以划分成两部分，高4bit 表示中断优先级别，低4bit 表示该中断在这一级别中的位置。LAPIC有两个寄存器用来处理优先级，TPR （task priority register）和 PPR \(Processor priority register\)。TPR用于确定当前CPU可处理的优先级别的范围。Task Priority一共4位（0～15），CPU只处理比TPR高的中断，低的则被pending到IRR中并不提交给CPU处理。TPR是由软件去读写的。



PPR处理器优先级寄存器。表示当前CPU正在处理的中断优先级别，并决定是否将Pending在IRR上的中断发给CPU。PPR是CPU硬件更新的。



设备中断传递大致流程如下\(假设设备接在IOAPIC IRQ3，而且PRT3的设置是 电平触发，低有效，Physical Mode APICID23H\)，Fixed Delivery Mode Vector为20H：

1. 设备通过拉低IRQ信号线产生一个中断。
2. 检测到低电平的有效信号之后，IOAPIC 把PRT03 中的Remote IRR\(Interrupt Request Register\) 和Delivery Status bit设置起来。
3. 使用PRT03中的信息，IOAPIC组成一个中断信息，使用写内存的方式发送给系统总线上。因为是APICID23H，所以LAPIC 23h将会识别这个中断信息是给它的，其他的Local APIC会忽略这个信息。
4. LAPIC 23h会把IRR中对应的bit设置起来（20h），表示Vector 20h有一个中断请求等待送给CPU。LAPIC会判断当前最高优先级的中断能否发送给CPU处理。
5. CPU清掉IRR 设置ISR表示中断开始被处理，处理完成以后写LAPIC EOI寄存器，LAPIC会清掉ISR。同时IOAPIC的EOI寄存器也会被写。
6. IOAPIC 收到EOI的写操作以后，它会比较PRT中的vector，在这里它会把PRT03的Remote IRR清零，之后它由可以识别IRQ3上的中断了。



