## Kubernetes网络模型 {#6j50r3}

* IP-per-Pod，每个Pod都拥有一个独立IP地址，Pod内所有容器共享一个网络命名空间
* 集群内所有Pod都在一个直接连通的扁平网络中，可通过IP直接访问
  * 所有容器之间无需NAT就可以直接互相访问
  * 所有Node和所有容器之间无需NAT就可以直接互相访问
  * 容器自己看到的IP跟其他容器看到的一样
* Service cluster IP尽可在集群内部访问，外部请求需要通过NodePort、LoadBalance或者Ingress来访问



容器的网络解决方案有很多种，每支持一种网络实现就进行一次适配显然是不现实的，而 CNI 就是为了兼容多种网络方案而发明的。CNI 是 Container Network Interface 的缩写，是一个标准的通用的接口，用于连接容器管理系统和网络插件。

简单来说，容器 runtime 为容器提供 network namespace，网络插件负责将 network interface 插入该 network namespace 中并且在宿主机做一些必要的配置，最后对 namespace 中的 interface 进行 IP 和路由的配置。

所以网络插件的主要工作就在于为容器提供网络环境，包括为 pod 设置 ip 地址、配置路由保证集群内网络的通畅。目前比较流行的网络插件是 Flannel 和 Calico。



