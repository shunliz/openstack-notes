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



Kubernetes 默认情况下就提供了主流的存储卷接入方案，我们可以执行命令 kubectl explain pod.spec.volumes 查看到支持的各种存储卷，另外也提供了插件机制，允许其他类型的存储服务接入到 Kubernetes 系统中来，在 Kubernetes 中就对应 In-Tree 和 Out-Of-Tree 两种方式，In-Tree 就是在 Kubernetes 源码内部实现的，和 Kubernetes 一起发布、管理的，但是更新迭代慢、灵活性比较差，Out-Of-Tree 是独立于 Kubernetes 的，目前主要有 CSI 和 FlexVolume 两种机制，开发者可以根据自己的存储类型实现不同的存储插件接入到 Kubernetes 中去，其中 CSI 是现在也是以后主流的方式。

概括说，k8s为了支持out-of-tree，设计了csi机制，并定制了一种行业标准接口规范，接下来主要内容就是它csi\(container storage interface\)。

# 3.存储架构和csi架构

k8s的存储结构图：![](/assets/compute-container-k8sdev-csi1.png)

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

用户实现的 CSI 插件，也就是CSI Driver存储驱动容器（正常和下面的CSI 插件是同一个程序，也可以分开做一个控制插件，一个操作插件）

与Master（kube-controller-manager）通信的辅助sidecar容器。

1. External Attacher：Kubernetes 提供的 sidecar 容器，它监听 VolumeAttachment 和 PersistentVolume 对象的变化情况，并调用 CSI 插件的 ControllerPublishVolume 和 ControllerUnpublishVolume 等 API 将 Volume 挂载或卸载到指定的 Node 上，也就是对应的 Attach/Detach 操作，因为 K8s 的 PV 控制器无法直接调用 Volume Plugins 的相关函数，故由 External Attacher 通过 gRPC 来调用。（官网提供）
2. External Provisioner：Kubernetes 提供的 sidecar 容器，它监听 PersistentVolumeClaim 对象的变化情况，并调用 CSI 插件的 ControllerPublish 和 ControllerUnpublish 等 API 管理 Volume，也就是Provision/Delete 操作，因为 K8s 的 PV 控制器无法直接调用 Volume Plugins 的相关函数，故由 External Provioner 通过 gRPC 来调用。（官网提供）

这两个容器通过本地Socket（Unix DomainSocket，UDS），并使用gPRC协议进行通信。

sidecar容器通过Socket调用CSI Driver容器的CSI接口，CSI Driver容器负责具体的存储卷操作。

* **左边一个Daemonset的pod**：对主机（Node）上的Volume进行管理和操作。在Kubernetes中建议将其部署为DaemonSet，在每个Node上都运行一个Pod，以便 Kubelet 可以调用，它包含 2 个容器：

* 用户实现的 CSI 插件，也就是CSI Driver存储驱动容器，主要功能是接收kubelet的调用，需要实现一系列与Node相关的CSI接口，例如NodePublishVolume接口（用于将Volume挂载到容器内的目标路径）、NodeUnpublishVolume接口（用于从容器中卸载Volume）等。

* node-driver-registrar：从宿主中暴露/var/lib/kubelet/plugins\_registry，挂载在容器的/registration，容器通过这个UDS向kubelet注册csi的UDS

* livenessprobe：可选

更详细的每个组件具体做什么，见参考\[7\]

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

# 5.实际操作来熟悉如何实现存储

pod创建过程中调用存储的相关流程

![](/assets/compute-container-k8s-csi4.png)

用户创建了一个包含 PVC 的 Pod，该 PVC 要求使用动态存储卷；

1. PV 控制器 watch 到该 Pod 使用的 PVC 处于 Pending 状态，于是调用 Volume Plugin（in-tree使用原生的Provisioner ，如果out-of-tree 由 External Provisioner 来处理）创建存储卷，并创建 PV 对象；
2. Scheduler 根据 Pod 配置、节点状态、PV 配置等信息，把 Pod 调度到一个合适的 Worker 节点上；
3. AD 控制器发现 Pod 和 PVC 处于待挂接状态，于是调用 Volume Plugin 挂接存储设备到目标 Worker 节点上
4. 在 Worker 节点上，Kubelet 中的 Volume Manager 等待存储设备挂接完成，并通过 Volume Plugin 将设备挂载到全局目录：/var/lib/kubelet/pods/\[pod uid\]/volumes/kubernetes.io~iscsi/\[PV name\]（以 iscsi 为例）；
5. Kubelet 通过 Docker 启动 Pod 的 Containers，用 bind mount 方式将已挂载到本地全局目录的卷映射到容器中。

原文链接：[https://blog.csdn.net/willinux20130812/article/details/120411540](https://blog.csdn.net/willinux20130812/article/details/120411540)

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

