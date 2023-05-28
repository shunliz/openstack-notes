![](/assets/network-pnet-netcard-multiq1.png)

**RSS、RPS、RFS**

RSS：网卡本身支持多队列时，可以直接使用RSS技术，直接由网卡的微处理器进行hash进行多队列处理，

RPS：在网卡驱动中根据数据包的四元组进行hash，此时cpu已经完成硬中断，而数据包也已经到达skb\_buffer，而完成硬中断的cpu核此时触发自身的软中断，并调用网卡驱动程序的注册函数，进行数据包四元组的hash计算，完成实际数据包的处理软中断的cpu核分配，也就是说，其实此时还在进行网卡数据包的硬中断的cpu处理核上增加了一次软中断，只是该调度核上面不再进行数据包协议栈的处理，涉及内核协议栈处理的过程的软中断交给了调度核。

RFS：虽然完成了数据包的软中断处理的核分配，但是如果，用户态的应用程序与内核协议栈处理过程的软中断使用的cpu核不一致，又会造成cpu cache的命中率降低，大大浪费了cpu的性能，因此需要给用户态的应用程序和对应flow中的数据包的cpu核处理为同一个，实现的过程就是给该应用程序hash一个id，使用该id去匹配对应的核，匹配的过程就是一个全局的rps\_sock\_flow\_table中有没有关于当前hash值对应的cpu核，如果有，就继续使用该cpu核，如果没有，就使用RPS分配的核，从而保证了应用程序与软中断使用了同一个cpu核。

![](/assets/network-pnet-netcard-multiq2.png)

![](/assets/network-pnet-netcard-multiq3.png)

## GRO（Generic Receive Offloading）

Large Receive Offloading \(LRO\) 是一个硬件优化，GRO 是 LRO 的一种软件实现。

两种方案的主要思想都是：通过合并“足够类似”的包来减少传送给网络栈的包数，这有助于减少 CPU 的使用量。例如，考虑大文件传输的场景，包的数量非常多，大部分包都是一段文件数据。相比于每次都将小包送到网络栈，可以将收到的小包合并成一个很大的包再送到网络栈。GRO 使协议层只需处理一个 header，而将包含大量数据的整个大包送到用户程序。

这类优化方式的缺点是信息丢失：包的 option 或者 flag 信息在合并时会丢失。这也是为什么大部分人不使用或不推荐使用LRO 的原因。

LRO 的实现，一般来说，对合并包的规则非常宽松。GRO 是 LRO 的软件实现，但是对于包合并的规则更严苛。如果用 tcpdump 抓包，有时会看到机器收到了看起来不现实的、非常大的包， 这很可能是系统开启了 GRO。

参考文献

[https://zhuanlan.zhihu.com/p/482772138?utm\_id=0](https://zhuanlan.zhihu.com/p/482772138?utm_id=0)

