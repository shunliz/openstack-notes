[https://www.cnblogs.com/shanghai1918/p/15526816.html](https://www.cnblogs.com/shanghai1918/p/15526816.html)

[https://baijiahao.baidu.com/s?id=1634216346397595390&wfr=spider&for=pc](https://baijiahao.baidu.com/s?id=1634216346397595390&wfr=spider&for=pc)

![](/assets/storage-ceph-optimize-pic1.png)

# ceph性能调优和最佳实践

1. 硬件选型

把握一个原则： TCO低，高性能，高可靠

一般企业使用ceph的历程： 硬件选型--部署调优--性能测试--架构灾备设计--部分业务上线测试--运行维护（故障处理，预案演练）



企业里典型的存储使用场景



（1）高性能场景： 低T CO下每秒拥有最高的IOPS



（2）通用场景：



（3）大容量场景：![](/assets/storage-ceph-optimize-pic2.png)![](/assets/storage-ceph-optimize-pic3.png)![](/assets/storage-ceph-optimize-pic4.png)![](/assets/storage-ceph-optimize-pic5.png)![](/assets/storage-ceph-optimize-pic6.png)

