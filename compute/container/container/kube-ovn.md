OpenStack + Kubernetes 是目前相对流行的云应用解决方案栈。对于那些部署了 OpenStack，同时又在尝试使用 Kubernetes 的用户，为了帮助他们更好地同时管理 OpenStack 和 K8s，我们建议用户以 K8s 作为底座，OpenStack 由 K8s 统一部署和管理，以及统一编排。各种租户子网（跨 K8s 和 OpenStack）能够动态定义和连接/断开，用以支持动态变化的租户需求。



在这种场景下，OpenStack VMs 和 Kubernetes Pods 的网络互通成为了一个亟待解决的问题。同时, OpenStack 所提供的网络隔离功能也应该在与 Kubernetes 连接的过程中生效: 属于一个 VPC 的 VMs 理所应当地不能访问另外一个 VPC 的 VMs 以及 Pods/Svcs。



面对这样的需求，灵雀云 Kube-OVN 团队工程师与 Intel 的 OpenStack 专家共同拟定了相应的方案。在方案里，Kube-OVN 需要提供的基本功能如下:



* 提供 Kubernetes Pods 和 OpenStack VMs 之间的路由交换, 实现网络互通；

* 为 Kubernetes 提供和 OpenStack 相同的网络隔离规范, 保证 VPC 之间的网络隔离。



目前有两类实现方案, 一种是基于 ovn-ic 的方案, 以松耦合的方式提供路由交换和网络隔离；另一种是基于同一个 ovn 底座的方案, 这是一种紧耦合的方案。

**方案前置要求：**



* Kube-OVN 升级到 V 1.7 版本以上；

* OpenStack usurri +. 必须基于 ovn 部署。



**方案一：基于 ovn-ic**

**组网方式一:**

ovn-ic 是 ovn 提供的一个 inter-connection 工具, 用于在不同的 ovn 控制器之间交换路由，可靠性和性能由 ovn 保证。

OpenStack + Kubernetes 是目前相对流行的云应用解决方案栈。对于那些部署了 OpenStack，同时又在尝试使用 Kubernetes 的用户，为了帮助他们更好地同时管理 OpenStack 和 K8s，我们建议用户以 K8s 作为底座，OpenStack 由 K8s 统一部署和管理，以及统一编排。各种租户子网（跨 K8s 和 OpenStack）能够动态定义和连接/断开，用以支持动态变化的租户需求。



在这种场景下，OpenStack VMs 和 Kubernetes Pods 的网络互通成为了一个亟待解决的问题。同时, OpenStack 所提供的网络隔离功能也应该在与 Kubernetes 连接的过程中生效: 属于一个 VPC 的 VMs 理所应当地不能访问另外一个 VPC 的 VMs 以及 Pods/Svcs。



面对这样的需求，灵雀云 Kube-OVN 团队工程师与 Intel 的 OpenStack 专家共同拟定了相应的方案。在方案里，Kube-OVN 需要提供的基本功能如下:



* 提供 Kubernetes Pods 和 OpenStack VMs 之间的路由交换, 实现网络互通；

* 为 Kubernetes 提供和 OpenStack 相同的网络隔离规范, 保证 VPC 之间的网络隔离。



目前有两类实现方案, 一种是基于 ovn-ic 的方案, 以松耦合的方式提供路由交换和网络隔离；另一种是基于同一个 ovn 底座的方案, 这是一种紧耦合的方案。

**方案前置要求：**



* Kube-OVN 升级到 V 1.7 版本以上；

* OpenStack usurri +. 必须基于 ovn 部署。



**方案一：基于 ovn-ic**

**组网方式一:**

ovn-ic 是 ovn 提供的一个 inter-connection 工具, 用于在不同的 ovn 控制器之间交换路由，可靠性和性能由 ovn 保证。![](/assets/compute-container-k8s-kubeovn1.png)

如图所示，ovn-ic 作为中间人, 相互交换 OpenStack OVN 和 Kubernetes Kube-OVN 的路由信息。

这样设计的好处是部署简单, OpenStack 和 Kubernetes 相对独立，互相之间不会影响。

同时也有一些弊端：OpenStack 和 Kubernetes 分别部署, 资源利用率独立计算；而且 OpenStack 的网络隔离特性无法展现。



**组网方式二:**

依然基于 ovn-ic, 但是会为 OpenStack 的每个租户建立一个 Kubernetes 集群。

![](/assets/compute-container-k8s-kubeovn2.png)

如图所示，ovn-ic 会负责打通 OpenStack 集群中的每个 VPC 网络和其对应的 Kubernetes 集群的路由。

这样设计的好处是部署简单, OpenStack 和 Kubernetes 相对独立, 互相不受对方变化的影响，同时能够支持 OpenStack 的网络隔离特性。

也有同样的弊端，OpenStack 和 Kubernetes 分别部署, 资源利用率独立计算。

**方案二：基于相同 OVN 底座**

这个方案要求 Kubernetes 与 OpenStack 基于相同的 ovn 控制器部署。Kube-OVN 目前支持 OpenStack 部署在 Kube-OVN 的 ovn 控制器上。

![](/assets/compute-container-k8s-kubeovn3.png)

在这个方案中，OpenStack 在部署时, 网络组件沿用了 Kubernetes 中的 Kube-OVN 提供的接口，因此，OpenStack 和 Kubernetes 的网络底座是同一个 ovn。相应地，Kubernetes 和 OpenStack 的组网能力也完全相同。

这个方案的好处显而易见：



* OpenStack 和 Kubernetes 能够在同一个 host node 部署 VM 和 Pod，能够最大化一个 host node 的资源利用率；

* 支持 OpenStack 的网络隔离特性。

当然，也会有一些弊端，包括部署复杂，Kubernetes 和 OpenStack 在虚拟网络组网时会有相互影响等。这两种方案已经登陆 GitHub（[https://github.com/kubeovn/kube-ovn/blob/master/docs/OpenstackOnKubernetes.md](https://xie.infoq.cn/link?target=https%3A%2F%2Fgithub.com%2Fkubeovn%2Fkube-ovn%2Fblob%2Fmaster%2Fdocs%2FOpenstackOnKubernetes.md)），哪种会更满足你的使用场景？



