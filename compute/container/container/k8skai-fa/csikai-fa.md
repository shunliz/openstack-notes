# 0.基础知识准备

[https://draveness.me/kubernetes-volume/](https://draveness.me/kubernetes-volume/)

[https://www.kubernetes.org.cn/4846.html](https://www.kubernetes.org.cn/4846.html)

[https://kingjcy.github.io/post/cloud/paas/base/kubernetes/k8s-store-csi/](https://kingjcy.github.io/post/cloud/paas/base/kubernetes/k8s-store-csi/)

# 1.名词解释

CSI是Container Storage Interface的简称，旨在能为容器编排引擎和存储系统间建立一套标准的存储调用接口，实现解耦，通过该接口能为容器编排引擎提供存储服务。

in-tree：代码逻辑在 K8s 官方仓库中；

out-of-tree：代码逻辑在 K8s 官方仓库之外，实现与 K8s 代码的解耦；

PV：PersistentVolume，集群级别的资源，由 集群管理员 or External Provisioner 创建。PV 的生命周期独立于使用 PV 的 Pod，PV 的 .Spec 中保存了存储设备的详细信息；

PVC：PersistentVolumeClaim，命名空间（namespace）级别的资源，由 用户 or StatefulSet 控制器（根据 VolumeClaimTemplate） 创建。PVC 类似于 Pod，Pod 消耗 Node 资源，PVC 消耗 PV 资源。Pod 可以请求特定级别的资源（CPU 和内存），而 PVC 可以请求特定存储卷的大小及访问模式（Access Mode）；

StorageClass：StorageClass 是集群级别的资源，由集群管理员创建。SC 为管理员提供了一种动态提供存储卷的“类”模板，SC 中的 .Spec 中详细定义了存储卷 PV 的不同服务质量级别、备份策略等等；

CSI：Container Storage Interface，目的是定义行业标准的“容器存储接口”，使存储供应商（SP）基于 CSI 标准开发的插件可以在不同容器编排（CO）系统中工作，CO 系统包括 Kubernetes、Mesos、Swarm 等。

# 2.存储原理

以csi-hostpath插件为例，演示部署CSI插件、用户使用CSI插件提供的存储资源。

## 开启csi {#开启csi}

设置Kubernetes服务启动参数，为kube-apiserver、kubecontroller-manager和kubelet服务的启动参数添加。

```
[root@k8smaster01 ~]# vi /etc/kubernetes/manifests/kube-apiserver.yaml
……
    - --allow-privileged=true
    - --feature-gates=CSIPersistentVolume=true
    - --runtime-config=storage.k8s.io/v1alpha1=true
……
[root@k8smaster01 ~]# vi /etc/kubernetes/manifests/kube-controller-manager.yaml
……
    - --feature-gates=CSIPersistentVolume=true
……
[root@k8smaster01 ~]# vi /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf
# Note: This dropin only works with kubeadm and kubelet v1.11+
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --feature-gates=CSIPersistentVolume=true"
……
[root@k8smaster01 ~]# systemctl daemon-reload
[root@k8smaster01 ~]# systemctl restart kubelet.service
```

Kubernetes 默认情况下就提供了主流的存储卷接入方案，我们可以执行命令 kubectl explain pod.spec.volumes 查看到支持的各种存储卷，另外也提供了插件机制，允许其他类型的存储服务接入到 Kubernetes 系统中来，在 Kubernetes 中就对应 In-Tree 和 Out-Of-Tree 两种方式，In-Tree 就是在 Kubernetes 源码内部实现的，和 Kubernetes 一起发布、管理的，但是更新迭代慢、灵活性比较差，Out-Of-Tree 是独立于 Kubernetes 的，目前主要有 CSI 和 FlexVolume 两种机制，开发者可以根据自己的存储类型实现不同的存储插件接入到 Kubernetes 中去，其中 CSI 是现在也是以后主流的方式。

概括说，k8s为了支持out-of-tree，设计了csi机制，并定制了一种行业标准接口规范，接下来主要内容就是它csi\(container storage interface\)。

# 3.存储架构和csi架构

k8s的存储结构图：![](/assets/compute-container-k8s-csi21.png)![](/assets/compute-container-k8sdev-csi1.png)

* PV Controller：负责 PV/PVC 的绑定，并根据需求进行数据卷的 Provision/Delete 操作
* AD Controller：负责存储设备的 Attach/Detach 操作，将设备挂载到目标节点
* Volume Manager：管理卷的 Mount/Unmount 操作、卷设备的格式化等操作
* Volume Plugin：扩展各种存储类型的卷管理能力，实现第三方存储的各种操作能力和 Kubernetes 存储系统结合

CSI 是由来自 Kubernetes、Mesos、 Cloud Foundry 等社区成员联合制定的一个行业标准接口规范，旨在将任意存储系统暴露给容器化应用程序。CSI 规范定义了存储提供商实现 CSI 兼容插件的最小操作集合和部署建议，CSI 规范的主要焦点是声明插件必须实现的接口。![](/assets/compute-container-k8s-csi2.png)

在 Kubernetes 上整合 CSI 插件的整体架构如上图所示：

Kubernetes CSI 存储体系主要由两部分组成：

* **Kubernetes 外部组件**：包含 Driver registrar、External provisioner、External attacher 三部分，这三个组件是从 Kubernetes 原本的 in-tree 存储体系中剥离出来的存储管理功能，实际上是 Kubernetes 中的一种外部 controller ，它们 watch kubernetes 的 API 资源对象，根据 watch 到的状态来调用下面提到的第二部分的 CSI 插件来实现存储的管理和操作。这部分是 Kubernetes 团队维护的，插件开发者完全不必关心其实现细节。

* Driver registra：用于将插件注册到 kubelet 的 sidecar 容器，并将驱动程序自定义的 NodeId 添加到节点的 Annotations 上，通过与 CSI 上面的 Identity 服务进行通信调用 CSI 的 GetNodeId 方法来完成该操作。

* External provisioner：用于 watch Kubernetes 的 PVC 对象并调用 CSI 的 CreateVolume 和 DeleteVolume 操作。

* External attacher：用于 Attach/Detach 阶段，通过 watch Kubernetes 的 VolumeAttachment 对象并调用 CSI 的 ControllerPublish 和 ControllerUnpublish 操作来完成对应的 Volume 的 Attach/Detach。而 Volume 的 Mount/Unmount 阶段并不属于外部组件，当真正需要执行 Mount 操作的时候，kubelet 会去直接调用下面的 CSI Node 服务来完成 Volume 的 Mount/UnMount 操作。

* **CSI 存储插件**: 这部分正是开发者需要实现的 CSI 插件部分，都是通过 gRPC 实现的服务，一般会用一个二进制文件对外提供服务，主要包含三部分：CSI Identity、CSI Controller、CSI Node。

* CSI Identity — 主要用于负责对外暴露这个插件本身的信息，确保插件的健康状态。

* CSI Controller - 主要实现 Volume 管理流程当中的 Provision 和 Attach 阶段，Provision 阶段是指创建和删除 Volume 的流程，而 Attach 阶段是指把存储卷附着在某个节点或脱离某个节点的流程，另外只有块存储类型的 CSI 插件才需要 Attach 功能。

* CSI Node — 负责控制 Kubernetes 节点上的 Volume 操作。其中 Volume 的挂载被分成了 NodeStageVolume 和 NodePublishVolume 两个阶段。NodeStageVolume 接口主要是针对块存储类型的 CSI 插件而提供的，块设备在 “Attach” 阶段被附着在 Node 上后，需要挂载至 Pod 对应目录上，但因为块设备在 linux 上只能 mount 一次，而在 kubernetes volume 的使用场景中，一个 volume 可能被挂载进同一个 Node 上的多个 Pod 实例中，所以这里提供了 NodeStageVolume 这个接口，使用这个接口把块设备格式化后先挂载至 Node 上的一个临时全局目录，然后再调用 NodePublishVolume 使用 linux 中的 bind mount 技术把这个全局目录挂载进 Pod 中对应的目录上。

![](/assets/compute-container-k8s-csi3.png)

这是csi官方推荐的部署方案。

* **右边一个StatefulSet或Deployment的pod**，可以说是csi controller，提供存储服务视角对存储资源和存储卷进行管理和操作。在Kubernetes中建议将其部署为单实例Pod，可以使用StatefulSet或Deployment控制器进行部署，设置副本数量为1，保证为一种存储插件只运行一个控制器实例。

  * 用户实现的 CSI 插件，也就是CSI Driver存储驱动容器（正常和下面的CSI 插件是同一个程序，也可以分开做一个控制插件，一个操作插件）

  * 与Master（kube-controller-manager）通信的辅助sidecar容器。

    * External Attacher：Kubernetes 提供的 sidecar 容器，它监听 VolumeAttachment 和 PersistentVolume 对象的变化情况，并调用 CSI 插件的 ControllerPublishVolume 和 ControllerUnpublishVolume 等 API 将 Volume 挂载或卸载到指定的 Node 上，也就是对应的 Attach/Detach 操作，因为 K8s 的 PV 控制器无法直接调用 Volume Plugins 的相关函数，故由 External Attacher 通过 gRPC 来调用。（官网提供）
    * External Provisioner：Kubernetes 提供的 sidecar 容器，它监听 PersistentVolumeClaim 对象的变化情况，并调用 CSI 插件的 ControllerPublish 和 ControllerUnpublish 等 API 管理 Volume，也就是Provision/Delete 操作，因为 K8s 的 PV 控制器无法直接调用 Volume Plugins 的相关函数，故由 External Provioner 通过 gRPC 来调用。（官网提供）

  * 这两个容器通过本地Socket（Unix DomainSocket，UDS），并使用gPRC协议进行通信。

  * sidecar容器通过Socket调用CSI Driver容器的CSI接口，CSI Driver容器负责具体的存储卷操作。

* **左边一个Daemonset的pod**：对主机（Node）上的Volume进行管理和操作。在Kubernetes中建议将其部署为DaemonSet，在每个Node上都运行一个Pod，以便 Kubelet 可以调用，它包含 2 个容器：

  * 用户实现的 CSI 插件，也就是CSI Driver存储驱动容器，主要功能是接收kubelet的调用，需要实现一系列与Node相关的CSI接口，例如NodePublishVolume接口（用于将Volume挂载到容器内的目标路径）、NodeUnpublishVolume接口（用于从容器中卸载Volume）等。

  * Driver Registrar：注册 CSI 插件到 kubelet 中，并初始化 NodeId（即给 Node 对象增加一个 Annotation csi.volume.kubernetes.io/nodeid）（官网提供）

  * node-driver-registrar容器与kubelet通过Node主机的一个hostPath目录下的unixsocket进行通信。CSI Driver容器与kubelet通过Node主机的另一个hostPath目录下的unixsocket进行通信，同时需要将kubelet的工作目录（默认为/var/lib/kubelet）挂载给CSIDriver容器，用于为Pod进行Volume的管理操作（包括mount、umount等）。

  * livenessprobe：可选

所以重点就是用户自己实现的插件逻辑，官方已经支持实现了很多的插件，我们在开发的时候可以参考。

上面简单的说了组件，以及相关的部署，下面我们通过一个交互图来了解如何实现out-of-tree的csi volume的。

右边的就是插件程序，由csi定义的三个部分组成，中间就是扩展的控制器，用于注册插件程序，联通集群和插件程序的交互的桥梁，左边就是我们的集群需要使用的组件，我们详细看一下。

* Driver Registrar 组件，负责将插件注册到 kubelet 里面（这可以类比为，将可执行文件放在插件目录下）。而在具体实现上，Driver Registrar 需要请求 CSI 插件的 Identity 服务来获取插件信息。
* External Provisioner 组件，负责的正是 Provision 阶段。在具体实现上，External Provisioner 监听（Watch）了 APIServer 里的 PVC 对象。当一个 PVC 被创建时，它就会调用 CSI Controller 的 CreateVolume 方法，为你创建对应 PV。
* External Attacher 组件，负责的正是“Attach 阶段”。在具体实现上，它监听了 APIServer 里  VolumeAttachment 对象的变化。VolumeAttachment 对象是 Kubernetes 确认一个 Volume 可以进入“Attach 阶段”的重要标志。一旦出现了 VolumeAttachment 对象，External Attacher 就会调用 CSI Controller 服务的 ControllerPublish 方法，完成它所对应的 Volume 的 Attach 阶段。
* Volume 的“Mount 阶段”，并不属于 External Components 的职责。当 kubelet 的 VolumeManagerReconciler 控制循环检查到它需要执行 Mount 操作的时候，会通过 pkg/volume/csi 包，直接调用 CSI Node 服务完成 Volume 的“Mount 阶段”。
  到这里我们就可以清晰的看到组件的作用，并且部署使用deployment和daemonset的含义，我们还可以通过网上的一个时序图来看看创建一个pod，各个组件之间的交互，更加详细的讲解了apiserver中的控制器是如何进行逻辑控制的。

![](/assets/compute-container-k8s-csi22.png)

调度流程

* 创建pod，使用pvc，然后就是正常的调度，选择好node后，对应的pvc添加annotation:volume.kubernetes.io/selected-node

provision流程

* 其实就是创建volume的流程，首先pv控制器watch 到该 Pod 使用的 PVC 处于 Pending 状态，查找集群中没有可以绑定的pv，动态创建。
* 动态创建使用的out-of-tree的模式，使用的csi场景下，这个时候通过external provisioner 来调用csi plugin ，使用调用csi对应的函数去创建pv对象，其实就是调用了Controller Plugin的CreateVolume函数。
* pv创建好后，就绑定pvc。
* 然后调度器将pod和node绑定。

attach流程

* 其实就是将pv挂载到node上，通过AD控制器watch pv，通过external attacher和csi plugin进行交互，调用Controller Plugin的ControllerPublishVolume函数完成attach操作。

mount流程

* 最后就是将pv挂载到pod中，kubelet中的volume manager调用csi plugin的NodeStageVolume、NodePublishVolume完成对应的mount操作，kubelet不需要使用grpc交互，直接调用本地二进制文件就可以。

说到这里，有必要说一下一个典型的 CSI Volume 生命周期如下图（来自 CSI SPEC）所示

![](/assets/compute-container-k8s-csi23.png)

Volume 被创建后进入 CREATED 状态，此时 Volume 仅在存储系统中存在，对于所有的 Node 或者 Container 都是不可感知的；

* 对 CREATED 状态的 Volume 进行 Controlller Publish 动作后在指定的 Node 上进入 NODE\_READY 的状态，此时 Node 可以感知到 Volume，但是 Container 依然不可见；
* 在 Node 对 Volume 进行 Stage 操作，进入 VOL\_READY 的状态，此时 Node 已经与 Volume 建立连接。Volume 在 Node 上以某种形式存在；
* 在 Node 上对 Volume 进行 Publish 操作，此时 Volume 将转化为 Node 上 Container 可见的形态被 Container 利用，进入正式服务的状态；
* 当 Container 的生命周期结束或其他不再需要 Volume 情况发生时，Node 执行 Unpublish Volume 动作，将 Volume 与 Container 之间的连接关系解除，回退至 VOL\_READY 状态；
* Node Unstage 操作将会把 Volume 与 Node 的连接断开，回退至 NODE\_READY 状态；
* Controller Unpublish 操作则会取消 Node 对 Volume 的访问权限；
* Delete 则从存储系统中销毁 Volume。

从这个图我们可以看出一个存储卷的供应分别调用了Controller Plugin的CreateVolume、ControllerPublishVolume及Node Plugin的NodeStageVolume、NodePublishVolume这4个gRPC接口，存储卷的销毁分别调用了Node Plugin的NodeUnpublishVolume、NodeUnstageVolume及Controller的ControllerUnpublishVolume、DeleteVolume这4个gRPC接口。

到此，csi的设计架构流程应该是很清晰的了，下面我们详细的看看其原理。



看到这里，应该清晰有哪些协助组件，和csidriver如何通信来实现可持久卷，下面是具体都有哪些grpc服务。

# 4.csidrvier开发需要实现的grpc服务

所谓的 CSI 插件开发事实上并非面向 Kubernetes API 开发，而是面向 Sidecar Containers 的 gRPC 开发，Sidecar Containers 一般会和我们自己开发的 CSI 驱动程序在同一个 Pod 中启动，然后 Sidecar Containers Watch API 中 CSI 相关 Object 的变动，接着通过本地 unix 套接字调用我们编写的 CSI 驱动。

sidecar containers指的就是协助组件。

官方说明在参考\[8\]

CSI定义了3套RPC接口，SP需要实现这3组接口。三组接口分别是：CSI Identity、CSI Controller 和 CSI Node，下面详细看看这些接口定义。

## 4.1 Identity Service

用于提供 CSI driver 的身份信息，Controller 和 Node 都需要实现。接口如下：

```
service Identity {
  rpc GetPluginInfo(GetPluginInfoRequest)
    returns (GetPluginInfoResponse) {}

  rpc GetPluginCapabilities(GetPluginCapabilitiesRequest)
    returns (GetPluginCapabilitiesResponse) {}

  rpc Probe (ProbeRequest)
    returns (ProbeResponse) {}
}
```

GetPluginInfo 是必须要实现的，node-driver-registrar 组件会调用这个接口将 CSI driver 注册到 kubelet；GetPluginCapabilities 是用来表明该 CSI driver 主要提供了哪些功能。

## 4.2 Controller Service

用于实现创建/删除 volume、attach/detach volume、volume 快照、volume 扩缩容等功能，Controller 插件需要实现这组接口。接口如下：

```
service Controller {
  rpc CreateVolume (CreateVolumeRequest)
    returns (CreateVolumeResponse) {}

  rpc DeleteVolume (DeleteVolumeRequest)
    returns (DeleteVolumeResponse) {}

  rpc ControllerPublishVolume (ControllerPublishVolumeRequest)
    returns (ControllerPublishVolumeResponse) {}

  rpc ControllerUnpublishVolume (ControllerUnpublishVolumeRequest)
    returns (ControllerUnpublishVolumeResponse) {}

  rpc ValidateVolumeCapabilities (ValidateVolumeCapabilitiesRequest)
    returns (ValidateVolumeCapabilitiesResponse) {}

  rpc ListVolumes (ListVolumesRequest)
    returns (ListVolumesResponse) {}

  rpc GetCapacity (GetCapacityRequest)
    returns (GetCapacityResponse) {}

  rpc ControllerGetCapabilities (ControllerGetCapabilitiesRequest)
    returns (ControllerGetCapabilitiesResponse) {}

  rpc CreateSnapshot (CreateSnapshotRequest)
    returns (CreateSnapshotResponse) {}

  rpc DeleteSnapshot (DeleteSnapshotRequest)
    returns (DeleteSnapshotResponse) {}

  rpc ListSnapshots (ListSnapshotsRequest)
    returns (ListSnapshotsResponse) {}

  rpc ControllerExpandVolume (ControllerExpandVolumeRequest)
    returns (ControllerExpandVolumeResponse) {}

  rpc ControllerGetVolume (ControllerGetVolumeRequest)
    returns (ControllerGetVolumeResponse) {
        option (alpha_method) = true;
    }
}
```

在上面介绍过 k8s 协助组件的时候已经提到，不同的接口分别提供给不同的组件调用，用于配合实现不同的功能。比如 CreateVolume/DeleteVolume 配合 external-provisioner 实现创建/删除 volume 的功能；ControllerPublishVolume/ControllerUnpublishVolume 配合 external-attacher 实现 volume 的 attach/detach 功能等。

## 4.3 Node Service

用于实现 mount/umount volume、检查 volume 状态等功能，Node 插件需要实现这组接口。接口如下：

```
service Node {
  rpc NodeStageVolume (NodeStageVolumeRequest)
    returns (NodeStageVolumeResponse) {}

  rpc NodeUnstageVolume (NodeUnstageVolumeRequest)
    returns (NodeUnstageVolumeResponse) {}

  rpc NodePublishVolume (NodePublishVolumeRequest)
    returns (NodePublishVolumeResponse) {}

  rpc NodeUnpublishVolume (NodeUnpublishVolumeRequest)
    returns (NodeUnpublishVolumeResponse) {}

  rpc NodeGetVolumeStats (NodeGetVolumeStatsRequest)
    returns (NodeGetVolumeStatsResponse) {}

  rpc NodeExpandVolume(NodeExpandVolumeRequest)
    returns (NodeExpandVolumeResponse) {}

  rpc NodeGetCapabilities (NodeGetCapabilitiesRequest)
    returns (NodeGetCapabilitiesResponse) {}

  rpc NodeGetInfo (NodeGetInfoRequest)
    returns (NodeGetInfoResponse) {}
}
```

NodeStageVolume 用来实现多个 pod 共享一个 volume 的功能，支持先将 volume 挂载到一个临时目录，然后通过 NodePublishVolume 将其挂载到 pod 中；NodeUnstageVolume 为其反操作。

# 5.基本原理

我们还是通过一个pod的创建过程中存储的相关操作来知悉如何实现存储，也是对上面核心流程的展开。

![](/assets/compute-container-k8s-csi4.png)

pod创建过程中调用存储的相关流程

* 用户创建了一个包含 PVC 的 Pod，该 PVC 要求使用动态存储卷；
* PV 控制器 watch 到该 Pod 使用的 PVC 处于 Pending 状态，于是调用 Volume Plugin（in-tree使用原生的Provisioner ，如果out-of-tree 由 External Provisioner 来处理）创建存储卷，并创建 PV 对象；
* Scheduler 根据 Pod 配置、节点状态、PV 配置等信息，把 Pod 调度到一个合适的 Worker 节点上；
* AD 控制器发现 Pod 和 PVC 处于待挂接状态，于是调用 Volume Plugin 挂接存储设备到目标 Worker 节点上
* 在 Worker 节点上，Kubelet 中的 Volume Manager 等待存储设备挂接完成，并通过 Volume Plugin 将设备挂载到全局目录：/var/lib/kubelet/pods/\[pod uid\]/volumes/kubernetes.io~iscsi/\[PV name\]（以 iscsi 为例）；
* Kubelet 通过 Docker 启动 Pod 的 Containers，用 bind mount 方式将已挂载到本地全局目录的卷映射到容器中。

## provisioning volumes {#provisioning-volumes}

先了解一些pv控制器的基本概念

![](/assets/compute-container-k8s-csi24.png)重图上可见pv控制器中主要有两个worker

* ClaimWorker：处理 PVC 的 add / update / delete 相关事件以及 PVC 的状态迁移；
* VolumeWorker：负责 PV 的状态迁移。

> PV 状态迁移（UpdatePVStatus）

* PV 初始状态为 Available，当 PV 与 PVC 绑定后，状态变为 Bound；
* 与 PV 绑定的 PVC 删除后，状态变为 Released；
* 当 PV 回收策略为 Recycled 或手动删除 PV 的 .Spec.ClaimRef 后，PV 状态变为 Available；
* 当 PV 回收策略未知或 Recycle 失败或存储卷删除失败，PV 状态变为 Failed；
* 手动删除 PV 的 .Spec.ClaimRef，PV 状态变为 Available。

> PVC 状态迁移（UpdatePVCStatus）

* 当集群中不存在满足 PVC 条件的 PV 时，PVC 状态为 Pending。在 PV 与 PVC 绑定后，PVC 状态由 Pending 变为 Bound；
* 与 PVC 绑定的 PV 在环境中被删除，PVC 状态变为 Lost；
* 再次与一个同名 PV 绑定后，PVC 状态变为 Bound。

再来看Provisioning 流程就十分清晰简单了

> 先寻找静态存储卷（FindBestMatch）

PV 控制器首先在环境中筛选一个状态为 Available 的 PV 与新 PVC 匹配。

* DelayBinding：PV 控制器判断该 PVC 是否需要延迟绑定：
  * 查看 PVC 的 annotation 中是否包含 volume.kubernetes.io/selected-node，若存在则表示该 PVC 已经被调度器指定好了节点（属于 ProvisionVolume），故不需要延迟绑定；
  * 若 PVC 的 annotation 中不存在 volume.kubernetes.io/selected-node，同时没有 StorageClass，默认表示不需要延迟绑定；若有 StorageClass，查看其 VolumeBindingMode 字段，若为 WaitForFirstConsumer 则需要延迟绑定，若为 Immediate 则不需要延迟绑定；
* FindBestMatchPVForClaim：PV 控制器尝试找一个满足 PVC 要求的环境中现有的 PV。PV 控制器会将所有的 PV 进行一次筛选，并会从满足条件的 PV 中选择一个最佳匹配的 PV。筛选规则：
  * VolumeMode 是否匹配；
  * PV 是否已绑定到 PVC 上；
  * PV 的 .Status.Phase 是否为 Available；
  * LabelSelector 检查，PV 与 PVC 的 label 要保持一致；
  * PV 与 PVC 的 StorageClass 是否一致；
  * 每次迭代更新最小满足 PVC requested size 的 PV，并作为最终结果返回；
* Bind：PV 控制器对选中的 PV、PVC 进行绑定：
  * 更新 PV 的 .Spec.ClaimRef 信息为当前 PVC；
  * 更新 PV 的 .Status.Phase 为 Bound；
  * 新增 PV 的 annotation ：pv.kubernetes.io/bound-by-controller: “yes”；
  * 更新 PVC 的 .Spec.VolumeName 为 PV 名称；
  * 更新 PVC 的 .Status.Phase 为 Bound；
  * 新增 PVC 的 annotation：pv.kubernetes.io/bound-by-controller: “yes” 和 pv.kubernetes.io/bind-completed: “yes”；

> 再动态创建存储卷（ProvisionVolume）

若环境中没有合适的 PV，则进入动态 Provisioning 场景：

* Before Provisioning：
  * PV 控制器首先判断 PVC 使用的 StorageClass 是 in-tree 还是 out-of-tree：通过查看 StorageClass 的 Provisioner 字段是否包含 “kubernetes.io/” 前缀来判断；
  * PV 控制器更新 PVC 的 annotation：
    `claim.Annotations[“volume.beta.kubernetes.io/storage-provisioner”] = storageClass.Provisioner`
    ；
* in-tree Provisioning（internal provisioning）：
  * in-tree 的 Provioner 会实现 ProvisionableVolumePlugin 接口的 NewProvisioner 方法，用来返回一个新的 Provisioner；
  * PV 控制器调用 Provisioner 的 Provision 函数，该函数会返回一个 PV 对象；
  * PV 控制器创建上一步返回的 PV 对象，将其与 PVC 绑定，Spec.ClaimRef 设置为 PVC，.Status.Phase 设置为 Bound，.Spec.StorageClassName 设置为与 PVC 相同的 StorageClassName；同时新增 annotation：“pv.kubernetes.io/bound-by-controller”=“yes” 和 “pv.kubernetes.io/provisioned-by”=plugin.GetPluginName\(\)；
* out-of-tree Provisioning（external provisioning）：
  * External Provisioner 检查 PVC 中的 claim.Spec.VolumeName 是否为空，不为空则直接跳过该 PVC；
  * External Provisioner 检查 PVC 中的
    `claim.Annotations[“volume.beta.kubernetes.io/storage-provisioner”]`
    是否等于自己的 Provisioner Name（External Provisioner 在启动时会传入 –provisioner 参数来确定自己的 Provisioner Name）；
  * 若 PVC 的 VolumeMode=Block，检查 External Provisioner 是否支持块设备；
  * External Provisioner 调用 Provision 函数：通过 gRPC 调用 CSI 存储插件的 CreateVolume 接口；
  * External Provisioner 创建一个 PV 来代表该 volume，同时将该 PV 与之前的 PVC 做绑定。

## deleting volumes {#deleting-volumes}

用户删除 PVC，PV 控制器改变 PV.Status.Phase 为 Released。

当 PV.Status.Phase == Released 时，PV 控制器首先检查 Spec.PersistentVolumeReclaimPolicy 的值，为 Retain 时直接跳过，为 Delete 时：

* in-tree Deleting：
  * in-tree 的 Provioner 会实现 DeletableVolumePlugin 接口的 NewDeleter 方法，用来返回一个新的 Deleter；
  * 控制器调用 Deleter 的 Delete 函数，删除对应 volume；
  * 在 volume 删除后，PV 控制器会删除 PV 对象；
* out-of-tree Deleting：
  * External Provisioner 调用 Delete 函数，通过 gRPC 调用 CSI 插件的 DeleteVolume 接口；
  * 在 volume 删除后，External Provisioner 会删除 PV 对象

## Attaching Volumes {#attaching-volumes}

Kubelet 组件和 AD 控制器都可以做 attach/detach 操作，若 Kubelet 的启动参数中指定了 –enable-controller-attach-detach，则由 Kubelet 来做；否则默认由 AD 控制起来做。

![](/assets/compute-container-k8s-csi25.png)AD 控制器中有两个核心变量：

* DesiredStateOfWorld（DSW）：集群中预期的数据卷挂接状态，包含了 nodes-&gt;volumes-&gt;pods 的信息；
* ActualStateOfWorld（ASW）：集群中实际的数据卷挂接状态，包含了 volumes-&gt;nodes 的信息。

Attaching 流程如下：

* AD 控制器根据集群中的资源信息，初始化 DSW 和 ASW。
* AD 控制器内部有三个组件周期性更新 DSW 和 ASW：
  * Reconciler。通过一个 GoRoutine 周期性运行，确保 volume 挂接 / 摘除完毕。此期间不断更新 ASW：
  * in-tree attaching：
    * in-tree 的 Attacher 会实现 AttachableVolumePlugin 接口的 NewAttacher 方法，用来返回一个新的 Attacher；
    * AD 控制器调用 Attacher 的 Attach 函数进行设备挂接；
    * 更新 ASW。
  * out-of-tree attaching：
    * 调用 in-tree 的 CSIAttacher 创建一个 VolumeAttachement（VA）对象，该对象包含了 Attacher 信息、节点名称、待挂接 PV 信息；
    * External Attacher 会 watch 集群中的 VolumeAttachement 资源，发现有需要挂接的数据卷时，调用 Attach 函数，通过 gRPC 调用 CSI 插件的 ControllerPublishVolume 接口。
  * DesiredStateOfWorldPopulator。通过一个 GoRoutine 周期性运行，主要功能是更新 DSW：
  * findAndRemoveDeletedPods - 遍历所有 DSW 中的 Pods，若其已从集群中删除则从 DSW 中移除；
  * findAndAddActivePods - 遍历所有 PodLister 中的 Pods，若 DSW 中不存在该 Pod 则添加至 DSW。
  * PVC Worker。watch PVC 的 add/update 事件，处理 PVC 相关的 Pod，并实时更新 DSW。

## Detaching Volumes {#detaching-volumes}

Detaching 流程如下：

* 当 Pod 被删除，AD 控制器会 watch 到该事件。首先 AD 控制器检查 Pod 所在的 Node 资源是否包含”volumes.kubernetes.io/keep-terminated-pod-volumes”标签，若包含则不做操作；不包含则从 DSW 中去掉该 volume；
* AD 控制器通过 Reconciler 使 ActualStateOfWorld 状态向 DesiredStateOfWorld 状态靠近，当发现 ASW 中有 DSW 中不存在的 volume 时，会做 Detach 操作：
  * in-tree detaching：
  * AD 控制器会实现 AttachableVolumePlugin 接口的 NewDetacher 方法，用来返回一个新的 Detacher；
  * 控制器调用 Detacher 的 Detach 函数，detach 对应 volume；
  * AD 控制器更新 ASW。
  * out-of-tree detaching：
  * AD 控制器调用 in-tree 的 CSIAttacher 删除相关 VolumeAttachement 对象；
  * External Attacher 会 watch 集群中的 VolumeAttachement（VA）资源，发现有需要摘除的数据卷时，调用 Detach 函数，通过 gRPC 调用 CSI 插件的 ControllerUnpublishVolume 接口；
  * AD 控制器更新 ASW。

## Mounting/Unmounting Volumes {#mounting-unmounting-volumes}

![](/assets/compute-container-k8s-csi26.png)Volume Manager 中同样也有两个核心变量：

* DesiredStateOfWorld（DSW）：集群中预期的数据卷挂载状态，包含了 volumes-&gt;pods 的信息；
* ActualStateOfWorld（ASW）：集群中实际的数据卷挂载状态，包含了 volumes-&gt;pods 的信息。

全局目录（global mount path）存在的目的：块设备在 Linux 上只能挂载一次，而在 K8s 场景中，一个 PV 可能被挂载到同一个 Node 上的多个 Pod 实例中。若块设备格式化后先挂载至 Node 上的一个临时全局目录，然后再使用 Linux 中的 bind mount 技术把这个全局目录挂载进 Pod 中对应的目录上，就可以满足要求。上述流程图中，全局目录即 /var/lib/kubelet/pods/\[pod uid\]/volumes/kubernetes.io~iscsi/\[PV name\]

Mounting/UnMounting 流程如下：

* VolumeManager 根据集群中的资源信息，初始化 DSW 和 ASW。
* VolumeManager 内部有两个组件周期性更新 DSW 和 ASW：
  * DesiredStateOfWorldPopulator：通过一个 GoRoutine 周期性运行，主要功能是更新 DSW；
  * Reconciler：通过一个 GoRoutine 周期性运行，确保 volume 挂载 / 卸载完毕。此期间不断更新 ASW：
  * unmountVolumes：确保 Pod 删除后 volumes 被 unmount。遍历一遍所有 ASW 中的 Pod，若其不在 DSW 中（表示 Pod 被删除），此处以 VolumeMode=FileSystem 举例，则执行如下操作：
    * Remove all bind-mounts：调用 Unmounter 的 TearDown 接口（若为 out-of-tree 则调用 CSI 插件的 NodeUnpublishVolume 接口）；
    * Unmount volume：调用 DeviceUnmounter 的 UnmountDevice 函数（若为 out-of-tree 则调用 CSI 插件的 NodeUnstageVolume 接口）；
    * 更新 ASW。
  * mountAttachVolumes：确保 Pod 要使用的 volumes 挂载成功。遍历一遍所有 DSW 中的 Pod，若其不在 ASW 中（表示目录待挂载映射到 Pod 上），此处以 VolumeMode=FileSystem 举例，执行如下操作：
    * 等待 volume 挂接到节点上（由 External Attacher or Kubelet 本身挂接）；
    * 挂载 volume 到全局目录：调用 DeviceMounter 的 MountDevice 函数（若为 out-of-tree 则调用 CSI 插件的 NodeStageVolume 接口）；
    * 更新 ASW：该 volume 已挂载到全局目录；
    * bind-mount volume 到 Pod 上：调用 Mounter 的 SetUp 接口（若为 out-of-tree 则调用 CSI 插件的 NodePublishVolume 接口）；
    * 更新 ASW。
  * unmountDetachDevices：确保需要 unmount 的 volumes 被 unmount。遍历一遍所有 ASW 中的 UnmountedVolumes，若其不在 DSW 中（表示 volume 已无需使用），执行如下操作：
    * Unmount volume：调用 DeviceUnmounter 的 UnmountDevice 函数（若为 out-of-tree 则调用 CSI 插件的 NodeUnstageVolume 接口）；
    * 更新 ASW。

# 6.参考

\[1\] [https://www.qikqiak.com/k8strain/storage/csi/](https://www.qikqiak.com/k8strain/storage/csi/)

\[2\] [https://www.infoq.cn/article/NIdj0c5AYN9VBZJvXfsA?utm\_source=related\_read\_bottom&utm\_medium=article](https://www.infoq.cn/article/NIdj0c5AYN9VBZJvXfsA?utm_source=related_read_bottom&utm_medium=article)

\[3\] [https://zhuanlan.zhihu.com/p/146321241](https://zhuanlan.zhihu.com/p/146321241)

\[4\] [https://kingjcy.github.io/post/cloud/paas/base/kubernetes/k8s-store-csi/](https://kingjcy.github.io/post/cloud/paas/base/kubernetes/k8s-store-csi/)

\[5\] [https://github.com/kubernetes/community/blob/master/contributors/design-proposals/storage/container-storage-interface.md](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/storage/container-storage-interface.md)

\[6\] [https://kubernetes-csi.github.io/docs/csi-driver-object.html](https://kubernetes-csi.github.io/docs/csi-driver-object.html)

\[7\] [https://blog.hdls.me/16255765577465.html](https://blog.hdls.me/16255765577465.html)

\[8\] [https://github.com/container-storage-interface/spec/blob/master/spec.md](https://github.com/container-storage-interface/spec/blob/master/spec.md)

\[9\] [https://mritd.com/2020/08/19/how-to-write-a-csi-driver-for-kubernetes/](https://mritd.com/2020/08/19/how-to-write-a-csi-driver-for-kubernetes/)

