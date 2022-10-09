octavia 作为openstack的负载均衡方案，实现4层和7层的负载，自Pike版本替换了neutron自带的LB方案（Neutorn LBaaS）。



**基本对象**

* loadbalancer：负载均衡器的服务对象，所有的配置、操作、客户端命令等等
* vip：作用在loadbalancer上，用于外部对内部的业务集群的访问入口
* listener：监听器对象就是一组负载规则，通过其配置外部对VIP访问的端口，算法，类型等等（haproxy的frontend 配置）
* pool：后端业务真实服务器集群（haproxy的backend配置）
* member：业务云服务器的统称（backend 配置中的一个 member）
* Health monitor ：周期性检查成员的状态，用于负载调度
* L7 Policy ：数据包转发动作
* L7 Rule ：数据包转发匹配域

![](/assets/network-virtualnet-neutron-octavia11.png)**基本架构**![](/assets/network-virtualnet-neutron-octavia2.png)**基本流程**

![](/assets/network-virtualnet-neutron-octavia3.png)参考：

https://www.cnblogs.com/liufarui/p/11209968.html



