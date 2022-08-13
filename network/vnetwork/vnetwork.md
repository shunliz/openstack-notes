# 虚拟网络



## OVN

* 架构![](/assets/ovn-architecture.png)
* Openstack/CMS plugin 是 CMS 和 OVN 的接口，将CMS 的配置转化成 OVN 的格式写到 Nnorthbound DB 。

* Northbound DB 存储逻辑数据，与传统网络设备概念一致，比如 logical switch，logical router，ACL，logical port。
* ovn-northd 类似于一个集中式控制器，把Northbound DB 里面的数据翻译后写到 Southbound DB 。
* Southbound DB 保存的数据和 Northbound DB 语义完全不一样，主要包含 3 类数据，一是物理网络数据，比如 HV（hypervisor）的 IP 地址，HV 的 tunnel 封装格式；二是逻辑网络数据，比如报文如何在逻辑网络里面转发；三是物理网络和逻辑网络的绑定关系，比如逻辑端口关联到哪个 HV 上面。
* ovn-controller 是 OVN 里面的 agent，类似于 neutron 里面的 ovs-agent，运行在每个 HV 上面，北向，ovn-controller 会把物理网络的信息写到 Southbound DB，南向，把 Southbound DB 保存的数据转化成 Openflow flow 配到本地的 OVS table 里面，来实现报文的转发。 
* ovs-vswitchd 和 ovsdb-server 是 OVS 的两个进程。

### 安装

apt-get update

apt-get -y install build-essential fakeroot

sudo apt-get install python-six openssl -y

sudo apt-get install openvswitch-switch openvswitch-common -y



sudo apt-get install ovn-central ovn-common ovn-host -y

sudo apt-get install ovn-host ovn-common -y



### 启动

/etc/init.d/openvswitch-switch start

/etc/init.d/ovn-central start

ovs-vsctl add-br br-int – set Bridge br-int fail-mode=secure



### 控制节点

ovn-nbctl set-connection ptcp:6641:172.18.22.197

ovn-sbctl set-connection ptcp:6642:172.18.22.197



ovs-vsctl set open . external-ids:ovn-remote=tcp:172.18.22.197:6642

ovs-vsctl set open . external-ids:ovn-encap-type=geneve

ovs-vsctl set open . external-ids:ovn-encap-ip=172.18.22.197



ovs-vsctl set open . external-ids:ovn-remote=tcp:172.18.22.197:6642

ovs-vsctl set open . external-ids:ovn-encap-type=geneve

ovs-vsctl set open . external-ids:ovn-encap-ip=172.18.22.198







