### TX卸载

tap设备默认开启TX卸载，让物理网卡计算从虚拟机出去的包的checksum，这部分是由openvswitch.ko处理。

### 用户空间datapath卸载

用户空间datapath\(dpdk\)不支持TX卸载，至少OVS 2.3-2.7是不支持的。如果OVS网桥开启TX卸载并使用用户空间datapath， 虚拟机连接到这个ovs不能建立TCP连接，L3和UDP没有问题。

当TCP第一个SYNC包到达虚拟机B， cksum incorrect然后被丢掉。原因是A启用了TCP TX offloading， 发往OVS的包使用的是假的checksum， 使用用户空间的datapath的ovs不做TCP checksum检查，数据包发给B，B收到包后校验TCP checksum，发现checksum不对，丢弃包。1s后重试，然后再次被丢弃。

![](/assets/network-virtualnet-ovs-codetx1.png)如果只关闭A的TX offloading，B发往A的包同样存在checksum不对问题，不能建立连接。同时关闭B的TCP TX offloading可以解决问题。

### 结论

用户空间datapath是高度优化的，并和TX offloading冲突（和dpdk vpp矢量数据包处理模式冲突）。启用dpdk的ovs需要开启RX offloading，关闭TX offloading。虚拟机自己计算cheksum。

