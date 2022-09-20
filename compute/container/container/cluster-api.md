### cluster-API

[http://gophercloud.io/](http://gophercloud.io/)Openstack GoSDK

[https://github.com/kubernetes-sigs/cluster-api](https://github.com/kubernetes-sigs/cluster-api)

一个用来简化K8S集群的部署，升级和多集群管理的框架。开放式框架，底层可插拔。支持裸机provisioning（metal3[https://metal3.io/](https://metal3.io/)）.

**cluster-api 可以理解成是一个集群生命周期管理的框架，因为它已经实现了一些基础的能力；可以理解成是集群生命周期管理的新的规范，因为它定义了集群生命周期的领域模型，也定义了新增一个自定义基础设施要支持 cluster-api 的实现标准。cluster-api 的领域模型主要包含**

![](/assets/compute-container-k8s-clusterapi1.png)

Cluster

Cluster 代表一个完整的集群的安装，包括安装 Kubernetes 集群需要的集群层面的资源，如 LB, VPC 等；包括机器的创建；暴露整个控制平面组件的安装；包括集群的健康检查等等。

BootstrapConfigBootstrapConfig 代表了集群的不同角色的机器在启动的时候的配置，如控制节点的 bootstrap config 和计算节点的 bootstrap config 是不同的，以及控制节点里的第一台机器和剩余的控制节点也是不同的。这需要注意的一点，BootstrapConfig 是和Machine 绑定的，不同的机器的 BootstrapConfig 是不同的。每个提供商在实现的时候可以完全不同，比如可以实现 kubeadm 的启动配置，也可以使用提供商自己封装好的启动配置 \(aws 就是自己实现的\)，所以可以理解成 bootstrap 其实是一种启动的类型定义，默认实现的 kubeadm 作为 bootstrap 的类型的，也可以自定义，比如像 aws 也是另外一种 bootstrap 的类型。

ControlPlaneControlPlane 代表了一个集群的控制平面的完整的安装，包括 api server，controller manager 等 Kubernetes 的管理组件。其中也包含了机器的创建，以及不同机器需要的 bootstrap config 的创建，集群证书的创建，集群控制平面的组件的健康检查，版本升级，控制平面的扩缩容等。一个 ControlPlane 被创建，同时状态变为 ready 的话，代表着一个 Kubernetes 的所有的管理组件都是 ready 的，可以直接使用 kubectl 去操作集群。

MachineMachine 代表了一台被集群纳管和使用的机器。不同的 provider 可以根据自己的平台实现自己怎样创建一台机器出来。这里有一点需要注意的是机器和启动配置的关系，每一台机器的启动都会绑定一个当前机器自己的 bootstrap config。

MachineDeploymentMachineDeployment 代表了一组机器的创建，类似于 Kubernetes 的 Deployment，可以通过 Deployment 创建出一组Pod出来。MachineDeployment 也是一样的逻辑，可以通过它创建一组机器出来。比如在集群里创建十台规模的计算节点出来，就是通过 MachineDeployment 来完成的。既然要创建一个 Kubernetes 的机器，那需要事先知道是哪个 provider 给我们创建机器，以及启动的 bootstrap config 是哪种 \(是不是 kubeadm 类型的 bootstrap\)，所以 MachineDeployment 的字段里就定义了相关的字段了，对应是 Bootstrap 和 Infrastructure 两个字段。

MachineSet

MachineSet 代表了一组机器的创建，类似于 Kubernetes 的 ReplicaSet，说到这里你应该知道它的作用了。

#### **cluster-api 整体架构**

![](/assets/compute-container-k8s-clusterapi2.png)

左侧部分中包含了 Cluster API，这部分是 cluster api 的标准组件，里面包含了作为一个标准框架需要提供的基础能力。比如 Cluster 的 Controller 实现；Control Plane 的 Controller 实现；MachineDeployment 的 Controller 的实现等等。

Bootstrap provider 是不同的厂商实现自己的启动配置，内置实现了基于 kubeadm 的启动配置。AWS 是使用自己的方式实现了 bootstrap config。

Infrastructure Provider 是不同的厂商实现自己在集群层面和机器层面所需要的资源的创建逻辑，比如 AWS 可以用 SDK 的方式创建机器，而 vSphere 使用基于虚拟机模版的方式创建机器。它代表了不同的厂商要遵循 cluster-api 的规范去实现的接口和逻辑。比如为集群创建 vpc，比如为控制平面创建虚拟机或者物理机。

Control Plane Provider 是不同的厂商自己实现自己的控制平面的方法，比如 cluster-api 内置实现了基于 kubeadm 方式的创建控制平面的方式，而 AWS 实现了基于 AWS 平台已有 Kubernetes 能力的 api 或者 SDK 的方式来实现控制平面的创建。

右侧部分中有一个 Custom Resources，这部分就是 cluster api 定义的接口标准，定义 Cluster 接口规范，定义了 Machine 是什么样的，定义了 ControlPlane 的接口规范，定义了 Bootstrap 的接口规范。而在提供商自己在实现的时候，会类似实现接口的方式实现自己的部分，以 Machine 为例，vSphere 会定义自己的 CRD \(VSphereMachine\)，以 Cluster 为例，vSphere 会定义自己的 CRD \(VSphereCluster\)。从这里就可以看出来接口的规范以及不同的提供商是怎样的实现接口规范的。

Target Cluster \(工作负载集群\) 是最终被创建出来的完整的 Kubernetes 集群。不同的提供商创建出来的 Kubernetes 有可能相同，也有可能不同。取决于自己的实现方式。比如说，以 kubeadm 作为 control plane 的实现，应该大致类似的；比如以 SDK 的方式直接利用 AWS 的基础设施能力应该就和 kubeadm 有很多不同的地方。

Management Cluster \(管理集群\) 是运行 cluster-api 组件，以及提供商提供的扩展组件的运行环境，类似 Cluster，Machine 等 CRD 对象就是保存在这个集群中的。

#### **cluster-api 原理解读之 Cluster 篇**

![](/assets/compute-container-k8s-clusterapi3.png)

总述：cluster-api 通过 cluster 这个 CRD，以及对应的 cluster controller 来完成 cluster 的整个逻辑的控制。主要的代码位于 controllers/cluster\_controller.go 和 controllers/cluster\_controller\_phases.go 两个文件中。通过分阶段处理的思路一步一步的处理安装一个集群的所需要的所有的步骤。这里的每一个步骤封装的比较好，所以要看每一个步骤具体的细节，就需要展开来看。以下给大家带来的是整体实现逻辑的步骤分解。逻辑分析的思路总结来源于源码。

1.watch Cluster/Machine，这里就是定义这个 controller 需要处理的 CRD 类型。

\2. Reconcile 方法主要做一些对象转换，以及判断存在不存在的常见操作

3.reconcile 方法是主要调和方法，主要是完成先为集群创建对应 provider 的资源，如 vpc，lb，等等，不会创建机器，只是创建一些集群层面需要的物理资源，这部分具体实现都是由 provider 自己实现，举例来说， AWSCluster 的 controller 的逻辑里就会去实现这部分。然后集群物理资源好的之后会设置 cluster 的 status 的 infra 的状态为 ready，接着调和 controlplane，这个过程最终是由 controlplane provider 的 controller 来实现的，目前的 controlplane 实现是 kubeadm，AWS 还有自己的实现；在集群的 controlplane 都启动好了，也就是 controlplane 的 status 也是 ready 的时候就会去拿这个 controlplane 控制的 Kubernetes 的集群的访问的 kubeconfig；最后通过检查所有属于 controlplane 是不是初始化了，如果没有就标记下 cluster 的 ControlPlaneInitializedCondition 为 “Waiting for the first control plane machine to have its status.nodeRef set”。

\4. 通过 SetControllerReference 和 Watch 机制来 watch 外部的资源对象，这里的外部是相对的，意思是对于 cluster 这个 CRD 以及 controller 而言的外部，比如 control plane 就是这里的外部的资源对象之一。然后通过更新外部对象触发外部对象的调和，同时由于 watch 了外部对象，cluster 对象也会因为外部对象的更新而触发自身的调和工作，当外部对象调和结束了会设置外部对象自身的 ready 为 true，这样 cluster 里面可以每次调和去检查外部对象的 status 里的 ready 是不是 ready，来判断基础设置的创建是不是 ready 的了。

\5. 以上方法负责触发外部对象的调和，根据外部对象的调和，再级联触发 cluster 的调和完成所有 cluster 创建所需要的做的所有工作，类似分布式系统的组件之间基于消息的通信。

\6. 调用 SetControllerReference，设置基础设施对象的 owner 是集群，同时设置 contrller 和 blockdelete 来帮助垃圾回收，以及后续要 watch 基础设施的能力。关键代码是在 OwnerReference 的 BlockOwnerDeletion 字段和 Controller 字段。

![](file://D:/code/gitee/cs-notes/images/Cloud/v2-de81bd589e6abad2f06caaa22f85a70b_720w.jpg?lastModify=1663690007 "img")

\7. 设置外部对象的 label，指定这个外部对象属于某一个集群，同时使用 patch 方法将变化部分更新到对象中，这样就会触发外部对象的更新事件从而触发外部对象对应的 controller 去调和这个外部对象。

\8. 让 cluster 的 controller 也去 watch 这个外部资源对象，这样当外部对象的数据变化时，也会触发 cluster 的调和。关键代码在 o.Controller.Watch\(\) 这里。

![](file://D:/code/gitee/cs-notes/images/Cloud/v2-c8805976dada853d561bdf886ef3c359_720w.jpg?lastModify=1663690007 "img")

\9. 先为集群创建对应 provider 的资源，如 vpc，lb，等等，不会创建机器，只是创建一些集群范围需要的物理资源，然后集群物理资源好了之后会设置 cluster 的 status 的 infra 的状态为 ready。实现的方式：是通过 controller-runtime 里面的方法设置 cluster 是 infrastructure 对象的 owner，同时通过 tracer 的 watch 方法让 cluster 的 controller 也会 watch infrastructure 对象的变化；这样 infrastructure 变化，cluster 也会进行调和动作。

\10. controlplane provider 的 controller 来实现的，目前的 controlplane 实现是 kubeadm，AWS 还有自己的实现,具体参看 controlplane 的 controller。实现的方式：是通过 controller-runtime 里面的方法设置 cluster 是 controlplane 对象的 owner，同时通过 tracer 的 watch 方法让 cluster 的 controller 也会 watch controlplane 对象的变化，这样 controlplane 变化，cluster 也会进行调和动作。

\11. 根据集群的信息去获取 kubeconfig，如果没有，就根据 cluster 对象的 ControlPlaneEndpoint，以及 cluster 的证书去生成一个这个 cluster 对象的 kubeconfig，保存到 secret 中，这个 secret 是以集群的 name 去命名的，代表这个集群的访问配置。在生成 kubeconfig 的过程中，需要用到的集群的 CA 证书，是通过直接查找 cluster-ca 的 secret 获得的，因为这个 CA 的证书，是 controlplane 负责给集群生成的，所以只需要查找对应的 secret 就可以了。

12.通过检查所有属于 controlplane 是不是初始化了，如果没有就标记一下cluster 的 ControlPlaneInitializedCondition 为 “Waiting for the first control plane machine to have its status.nodeRef “。关键代码就在以下截图中，有一个节点满足就会返回。意思就是有任何一台控制平面的机器，然后机器的对应的 Kubernetes 的 node 信息是不为空的 \(也就是属于 Kubernetes 的正常节点\)。

![](file://D:/code/gitee/cs-notes/images/Cloud/v2-d8d982a3ffd4f7ae497bfee9667317c5_720w.jpg?lastModify=1663690007 "img")

