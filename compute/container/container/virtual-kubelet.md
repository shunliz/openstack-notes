### virtual kubelet

Virtual Kubelet 最初是由微软 Azure 发起的开源项目，目标是让公共云的弹性容器实例类产品能与 Kubernetes 更好地集成，实现 Kubernetes 的 Serverless 能力。在实现上， Virtual Kubelet 提供了一种机制，可以与多家不同的供应商集成，如 Azure 的 ACI、AWS 的 fargate、华为的 CCI。 Virutal Kubelet 启动时会伪装成一个Kubelet 计算节点，即虚拟节点（Virtual Node），向 Kubernetes API Server 发起注册，并持续监听 Pod 变化事件。当Kubernetes 将Pod 调度到虚拟节点上时，Virtual Kubelet 会调用provider 的API 接口，将创建请求动态转发给底层的provider 的API 接口。在阿里云的Serverless Kubernetes 中， Virtual Kubelet 是Serverless Kubernetes 和 ECI 的连接器。如下图所示。

![](/assets/compute-container-k8s-virtualkubelete1.png)

### tensile kube介绍

下面我们以tensile kube为例子，熟悉一下Virtual Kubelet，tensile kube是腾讯游戏Tenc容器团队对外开源的K8s多集群调度方案，其将一个k8s集群作为一个Virtual Node，添加到主k8s集群中，这样当主集群的调度器就可以绑定Pod到Virtual Node（也就是子k8s集群）上。

对公司来说，tensile kube能够提供如下收益：

集群碎片资源整合，对于离线集群来说，或多或少存在资源碎片。通过tensile-kube，可以将这写碎片合成一个大的资源池，将闲置的资源充分利用起来。

服务多集群调度，结合Affinity等，可以实现Pod的多集群调度。对于1.16及以上集群还可以基于TopologySpreadConstraint实现更细粒度的跨级群调度。

服务跨集群通信，tensile-kube原生提供通过Service进行跨集群通信的能力。但是前提是：需要不同集群之间的网络通过Pod IP可以互通\(开启ServiceControllers\)，如：私有集群共用一个Flannel等。

便捷化运维集群，当我们进行单个集群升级、变更时，往往需要通知业务迁移、集群屏蔽调度等，在tensile-kube的支持，我们只需要将virtual-node设置为不可能调度。

原生kubectl能力，tensile kubectl支持原生kubectl所有操作，包括：kuebctl logs和kubectl exec。这对于运维人员去管理对于下层集群上的Pod会十分方便，没有额外的学习成本。

  
![](/assets/compute-container-k8s-virutalkubelet2.png)

