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



