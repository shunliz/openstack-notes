## Kubernetes网络模型 {#6j50r3}

* IP-per-Pod，每个Pod都拥有一个独立IP地址，Pod内所有容器共享一个网络命名空间
* 集群内所有Pod都在一个直接连通的扁平网络中，可通过IP直接访问
  * 所有容器之间无需NAT就可以直接互相访问
  * 所有Node和所有容器之间无需NAT就可以直接互相访问
  * 容器自己看到的IP跟其他容器看到的一样
* Service cluster IP尽可在集群内部访问，外部请求需要通过NodePort、LoadBalance或者Ingress来访问



