**概述**

下面的分析是k8s通过ceph-csi（csi plugin）接入ceph存储（csi相关组件的分析以rbd为例进行分析），对csi系统结构、所涉及的k8s对象与组件进行了简单的介绍，以及k8s对存储进行相关操作的流程分析，存储相关操作包括了存储创建、存储扩容、存储挂载、解除存储挂载以及存储删除操作。

**csi系统结构**

这是一张k8s csi的系统架构图，图中所画的组件以及k8s对象，接下来会一一进行分析。

![](/assets/compute-container-k8s-cephcsi11.png)**csi简介**

CSI是Container Storage Interface（容器存储接口）的简写。

CSI的目的是定义行业标准“容器存储接口”，使存储供应商（SP）能够开发一个符合CSI标准的插件并使其可以在多个容器编排（CO）系统中工作。CO包括Cloud Foundry, Kubernetes, Mesos等。

CSI组件一般采用容器化部署，减少了环境依赖。

**涉及k8s对象**

1. **PersistentVolume**

持久存储卷，集群级别资源，代表了存储卷资源，记录了该存储卷资源的相关信息。

回收策略

（1）retain：保留策略，当删除PVC的时候，PV与外部存储资源仍然存在。

（2）delete：删除策略，当与pv绑定的pvc被删除的时候，会从k8s集群中删除PV对象，并执行外部存储资源的删除操作。

（3）resycle（已废弃）

pv状态迁移

available --&gt; bound --&gt; released

1. **PersistentVolumeClaim**

持久存储卷声明，namespace级别资源，代表了用户对于存储卷的使用需求声明。

示例：

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test
  namespace: test
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  storageClassName: csi-cephfs-sc
  volumeMode: Filesystem
```

###### pvc状态迁移

pending --&gt; bound

### 3. StorageClass

定义了创建pv的模板信息，集群级别资源，用于动态创建pv。

示例：

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: csi-rbd-sc
parameters:
  clusterID: ceph01
  imageFeatures: layering
  imageFormat: "2"
  mounter: rbd
  pool: kubernetes
provisioner: rbd.csi.ceph.com
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

**4. VolumeAttachment**

VolumeAttachment 记录了pv的相关挂载信息，如挂载到哪个node节点，由哪个volume plugin来挂载等。

AD Controller 创建一个 VolumeAttachment，而 External-attacher 则通过观察该 VolumeAttachment，根据其状态属性来进行存储的挂载和卸载操作。

示例：

```
apiVersion: storage.k8s.io/v1
kind: VolumeAttachment
metadata:
  name: csi-123456
spec:
  attacher: cephfs.csi.ceph.com
  nodeName: 192.168.1.10
  source:
    persistentVolumeName: pvc-123456
status:
  attached: true
```

**5. CSINode**

CSINode 记录了csi plugin的相关信息（如nodeId、driverName、拓扑信息等）。

当Node Driver Registrar向kubelet注册一个csi plugin后，会创建（或更新）一个CSINode对象，记录csi plugin的相关信息。

示例：

```
apiVersion: storage.k8s.io/v1
kind: CSINode
metadata:
  name: 192.168.1.10
spec:
  drivers:
  - name: cephfs.csi.ceph.com
    nodeID: 192.168.1.10
    topologyKeys: null
  - name: rbd.csi.ceph.com
    nodeID: 192.168.1.10
    topologyKeys: null
```

**涉及组件与作用**

下面先简单介绍下涉及的组件与作用，后面会有单独详细的介绍各个组件的作用。

**1. volume plugin**

扩展各种存储类型的卷的管理能力，实现第三方存储的各种操作能力与k8s存储系统的结合。调用第三方存储的接口或命令，从而提供数据卷的创建/删除、attach/detach、mount/umount的具体操作实现，可以认为是第三方存储的代理人。前面分析组件中的对于数据卷的创建/删除、attach/detach、mount/umount操作，全是调用volume plugin来完成。

后续对volume plugin的详细分析，以通过ceph-csi操作rbd为例进行分析。

根据源码所在位置，volume plugin分为in-tree与out-of-tree。

**in-tree**

在k8s源码内部实现，和k8s一起发布、管理，更新迭代慢、灵活性差。

**out-of-tree**

代码独立于k8s，由存储厂商实现，有csi、flexvolume两种实现。

**csi plugin**

本次的分析为k8s通过ceph-csi来使用ceph存储，ceph-csi属于csi plugin。csi plugin分为ControllerServer与NodeServer，各负责不同的存储操作。

**external plugin**

external plugin包括了external-provisioner、external-attacher、external-resizer、external-snapshotter等，external plugin辅助csi plugin组件，共同完成了存储相关操作。external plugin负责watch pvc、volumeAttachment等对象，然后调用volume plugin来完成存储的相关操作。如external-provisioner watch pvc对象，然后调用csi plugin来创建存储，最后创建pv对象；external-attacher watch volumeAttachment对象，然后调用csi plugin来做attach/dettach操作；external-resizer watch pvc对象，然后调用csi plugin来做存储的扩容操作等。

**Node-Driver-Registrar**

Node-Driver-Registrar组件负责实现csi plugin（NodeServer）的注册，让kubelet感知csi plugin的存在。

组件部署方式

csi plugin controllerServer与external plugin作为容器，使用deployment部署，多副本可实现高可用；而csi plugin NodeServer与Node-Driver-Registrar作为容器，使用daemonset部署，即每个node节点都有。

**2. kube-controller-manager**

**PV controller**

负责pv、pvc的绑定与生命周期管理（如创建/删除底层存储，创建/删除pv对象，pv与pvc对象的状态变更）。

创建/删除底层存储、创建/删除pv对象的操作，由PV controller调用volume plugin（in-tree）来完成。本次分析的是k8s通过ceph-csi来使用ceph存储，volume plugin为ceph-csi，属于out-tree，所以创建/删除底层存储、创建/删除pv对象的操作由external-provisioner来完成。

**AD controller**

AD Cotroller全称Attachment/Detachment 控制器，主要负责创建、删除VolumeAttachment对象，并调用volume plugin来做存储设备的Attach/Detach操作（将数据卷挂载到特定node节点上/从特定node节点上解除挂载），以及更新node.Status.VolumesAttached等。

不同的volume plugin的Attach/Detach操作逻辑有所不同，如通过ceph-csi（out-tree volume plugin）来使用ceph存储，则的Attach/Detach操作只是修改VolumeAttachment对象的状态，而不会真正的将数据卷挂载到节点/从节点上解除挂载，真正的节点存储挂载/解除挂载操作由kubelet中volume manager调用rc.operationExecutor.MountVolume/rc.operationExecutor.UnmountDevice方法时，调用ceph-csi来完成，后面会有博文详细做介绍。

**3. kubelet**

**volume manager**

主要是管理卷的Attach/Detach（与AD controller作用相同，通过kubelet启动参数控制哪个组件来做该操作，后续会详细介绍）、mount/umount等操作。

本次的分析为k8s通过ceph-csi来使用ceph存储。本次分析中，volume manager的Attach/Detach操作只创建/删除VolumeAttachment对象，而不会真正的将数据卷挂载到节点/从节点上解除挂载；csi-attacer组件也不会做挂载/解除挂载操作，只是更新VolumeAttachment对象，真正的节点挂载/解除挂载操作由kubelet中volume manager调用rc.operationExecutor.MountVolume/rc.operationExecutor.UnmountDevice方法时，调用ceph-csi来完成，后面会有博文详细做介绍。

**kubernetes创建与挂载volume（in-tree volume plugin）**

先来看下kubernetes通过in-tree volume plugin来创建与挂载volume的流程

![](/assets/compute-conatiner-k8s-cephcsi113.png)

（1）用户创建pvc；

（2）PV controller watch到pvc的创建，寻找合适的pv与之绑定。

（3）（4）当找不到合适的pv时，将调用volume plugin来创建volume，并创建pv对象，之后该pv对象与pvc对象绑定。

（5）用户创建挂载pvc的pod；

（6）kube-scheduler watch到pod的创建，为其寻找合适的node调度。

（7）（8）pod调度完成后，AD controller/volume manager watch到pod声明的volume没有进行attach操作，将调用volume plugin来做attach操作。

（9）volume plugin进行attach操作，将volume挂载到pod所在node节点，成为如/dev/vdb的设备。

（10）（11）attach操作完成后，volume manager watch到pod声明的volume没有进行mount操作，将调用volume plugin来做mount操作。

（12）volume plugin进行mount操作，将node节点上的第（9）步得到的/dev/vdb设备挂载到指定目录。

**kubernetes创建与挂载volume（out-of-tree volume plugin）**

再来看下kubernetes通过out-of-tree volume plugin来创建与挂载volume的流程，以csi-plugin为例。

![](/assets/compute-container-k8s-cephcsi112.png)（1）用户创建pvc；

（2）PV controller watch到pvc的创建，寻找合适的pv与之绑定。当寻找不到合适的pv时，将更新pvc对象，添加annotation：volume.beta.kubernetes.io/storage-provisioner，让external-provisioner组件开始开始创建存储与pv对象的操作。

（3）external-provisioner组件watch到pvc的创建，判断annotation：volume.beta.kubernetes.io/storage-provisioner的值，即判断是否是自己来负责做创建操作，是则调用csi-plugin ControllerServer来创建存储，并创建pv对象。

（4）PV controller watch到pvc，寻找合适的pv（上一步中创建）与之绑定。

（5）用户创建挂载pvc的pod；

（6）kube-scheduler watch到pod的创建，为其寻找合适的node调度。

（7）（8）pod调度完成后，AD controller/volume manager watch到pod声明的volume没有进行attach操作，将调用csi-attacher来做attach操作（实际上只是创建volumeAttachement对象）。

（9）external-attacher组件watch到volumeAttachment对象的新建，调用csi-plugin进行attach操作（如果volume plugin是ceph-csi，external-attacher组件watch到volumeAttachment对象的新建后，只是修改该对象的状态属性，不会做attach操作，真正的attach操作由kubelet中的volume manager调用volume plugin ceph-csi来完成）。

（10）csi-plugin ControllerServer进行attach操作，将volume挂载到pod所在node节点，成为如/dev/vdb的设备。

（11）（12）attach操作完成后，volume manager watch到pod声明的volume没有进行mount操作，将调用csi-mounter来做mount操作。

（13）csi-mounter调用csi-plugin NodeServer进行mount操作，将node节点上的第（10）步得到的/dev/vdb设备挂载到指定目录。

**kubernetes存储相关操作流程具体分析（out-of-tree volume plugin，以ceph-csi为例）**

下面来看下kubernetes通过out-of-tree volume plugin来创建/删除、挂载/解除挂载volume的流程。

下面先对每个操作的整体流程进行分析，后面会对涉及的每个组件进行源码分析。

**1. 存储创建**

**流程图      
**![](/assets/compute-container-k8s-cephcsi114.png)**流程分析**

（1）用户创建pvc对象；

（2）pv controller监听pvc对象，寻找现存的合适的pv对象，与pvc对象绑定。当找不到现存合适的pv对象时，将更新pvc对象，添加annotation：volume.beta.kubernetes.io/storage-provisioner，让external-provisioner组件开始开始创建存储与pv对象的操作；当找到时，将pvc与pv绑定，结束操作。

（3）external-provisioner组件监听到pvc的新增事件，判断pvc的annotation：volume.beta.kubernetes.io/storage-provisioner的值，即判断是否是自己来负责做创建操作，是则调用ceph-csi组件进行存储的创建；

（4）ceph-csi组件调用ceph创建底层存储；

（5）底层存储创建完成后，external-provisioner根据存储信息，拼接pv对象，创建pv对象；

（6）pv controller监听pvc对象，寻找合适的pv对象，与pvc对象绑定。

**2. 存储扩容**

**流程图      
**![](/assets/compute-container-k8s-cephcsi115.png)

**流程分析**

（1）修改pvc对象，修改申请存储大小（pvc.spec.resources.requests.storage）；

（2）修改成功后，external-resizer监听到该pvc的update事件，发现pvc.Spec.Resources.Requests.storgage比pvc.Status.Capacity.storgage大，于是调ceph-csi组件进行 controller端扩容；

（3）ceph-csi组件调用ceph存储，进行底层存储扩容；

（4）底层存储扩容完成后，external-resizer组件更新pv对象的.Spec.Capacity.storgage的值为扩容后的存储大小；

（5）kubelet的volume manager在reconcile\(\)调谐过程中发现pv.Spec.Capacity.storage大于pvc.Status.Capacity.storage，于是调ceph-csi组件进行 node端扩容；

（6）ceph-csi组件对dnode上存储对应的文件系统扩容；

（7）扩容完成后，kubelet更新pvc.Status.Capacity.storage的值为扩容后的存储大小。

**3. 存储挂载**

**流程图**

kubelet启动参数–enable-controller-attach-detach，该启动参数设置为 true 表示启用 Attach/Detach controller进行Attach/Detach 操作，同时禁用 kubelet 执行 Attach/Detach 操作（默认值为 true）。实际上Attach/Detach 操作就是创建/删除VolumeAttachment对象。

（1）kubelet启动参数–enable-controller-attach-detach=true，Attach/Detach controller进行Attach/Detach 操作

![](/assets/compute-container-k8s-cephcsi116.png)（2）kubelet启动参数–enable-controller-attach-detach=false，kubelet端volume manager进行Attach/Detach 操作

![](/assets/compute-container-k8s-cephsci117.png)流程分析

（1）用户创建一个挂载了pvc的pod；

（2）AD controller或volume manager中的reconcile\(\)发现有volume未执行attach操作，于是进行attach操作，即创建VolumeAttachment对象；

（3）external-attacher组件list/watch VolumeAttachement对象，更新VolumeAttachment.status.attached=true；

（4）AD controller更新node对象的.Status.VolumesAttached属性值，将该volume记为attached；

（5）kubelet中的volume manager获取node.Status.VolumesAttached属性值，发现volume已被标记为attached；

（6）于是volume manager中的reconcile\(\)调用ceph-csi组件的NodeStageVolume与NodePublishVolume完成挂载。

**4. 解除存储挂载**

**流程图**

（1）AD controller

![](/assets/compute-container-k8s-cephcsi118.png)（2）volume manager

![](/assets/compute-container-k8s-cephcsi119.png)流程分析

（1）用户删除声明了pvc的pod；



（2）AD controller或volume manager中的reconcile\(\)发现有volume未执行dettach操作，于是进行dettach操作，即删除VolumeAttachment对象；



（3）AD controller或volume manager等待VolumeAttachment对象删除成功；



（4）AD controller更新新node对象的.Status.VolumesAttached属性值，将标记为attached的该volume从属性值中去除；



（5）kubelet中的volume manager获取node.Status.VolumesAttached属性值，找不到相关的volume信息；



（6）于是volume manager中的reconcile\(\)调用ceph-csi组件的NodeUnpublishVolume与NodeUnstageVolume完成解除挂载。



**5. 删除存储**

**流程图**

![](/assets/compute-container-k8s-cephcsi1110.png)流程分析

（1）用户删除pvc对象；



（2）pv controller发现与pv绑定的pvc对象被删除，于是更新pv的状态为released；



（3）external-provisioner watch到pv更新事件，并检查pv的状态是否为released，以及回收策略是否为delete；



（4）接下来external-provisioner组件会调用ceph-csi的DeleteVolume来删除存储；



（5）ceph-csi组件的DeleteVolume方法，调用ceph集群命令，删除底层存储；



（6）external-provisioner组件删除pv对象。



