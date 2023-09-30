# OVS架构

![](/assets/network-virtualnet-ovs-code1.png)vswitchd

* * 用户空间程序，后台进程
  * 工具:
    `ovs-appctl`
* ovsdb-server
  * 用户空间程序，ovs数据库服务
  * 工具:
    `ovs-vsctl`,`ovs-ofctl`
* 内核模块 \(datapath\)
  * 内核模块，包转发工具
  * 工具:
    `ovs-dpctl`



