# kube-controller-manager之AD Cotroller源码分析 {#articleContentId}

**概述**

kube-controller-manager组件中，有两个controller与存储相关，分别是PV controller与AD controller。



基于tag v1.17.4

https://github.com/kubernetes/kubernetes/releases/tag/v1.17.4



**AD Cotroller分析**

**AD Cotroller作用**

AD Cotroller全称Attachment/Detachment 控制器，主要负责创建、删除VolumeAttachment对象，并调用volume plugin来做存储设备的Attach/Detach操作（将数据卷挂载到特定node节点上/从特定node节点上解除挂载），以及更新node.Status.VolumesAttached等。



不同的volume plugin的Attach/Detach操作逻辑有所不同，如通过ceph-csi（out-tree volume plugin）来使用ceph存储，则的Attach/Detach操作只是修改VolumeAttachment对象的状态，而不会真正的将数据卷挂载到节点/从节点上解除挂载。



kubelet启动参数–enable-controller-attach-detach

AD Cotroller与kubelet中的volume manager逻辑相似，都可以做Attach/Detach操作，但是kube-controller-manager与kubelet中，只会有一个组件做Attach/Detach操作，通过kubelet启动参数–enable-controller-attach-detach设置。设置为 true 表示启用kube-controller-manager的AD controller来做Attach/Detach操作，同时禁用 kubelet 执行 Attach/Detach 操作（默认值为 true）。



**注意**

当k8s通过ceph-csi来使用ceph存储，volume plugin为ceph-csi，AD controller的Attach/Detach操作，只是创建/删除VolumeAttachment对象，而不会真正的将数据卷挂载到节点/从节点上解除挂载；csi-attacer组件也不会做挂载/解除挂载操作，只是更新VolumeAttachment对象，真正的节点挂载/解除挂载操作由kubelet中的volume manager调用volume plugin（ceph-csi）来完成。



**两个关键结构体**

（1）desiredStateOfWorld: 记录着集群中期望要挂载到node的pod的volume信息，简称DSW，数据来源：pod里声明的volume列表。



（2）actualStateOfWorld: 记录着集群中实际已经挂载到node节点的volume信息，简称ASW，数据来源：node对象中的node.Status.VolumesAttached属性值列表。



**node.Status.VolumesInUse作用**

node对象中的node.Status.VolumesInUse记录的是已经attach到该node节点上，并已经mount了的volume信息。



该属性由kubelet的volume manager来更新。当一个volume加入dsw中，就会被更新到node.Status.VolumesInUse中，直到该volume在asw与dsw中均不存在或已经处于unmounted状态时，会从node.Status.VolumesInUse中移除。



AD controller做volume的dettach操作前，会先判断该属性，如果该属性值中含有该volume，则说明volume还在被使用，返回dettach失败的错误。



**node.Status.VolumesAttached作用**

node对象中的node.Status.VolumesAttached记录的是已经attach到该node节点上的volume信息。



该属性由kube-controller-manager中的AD controller根据asw的值来更新。



**attachDetachController.Run源码分析**

直接看到attachDetachController的Run\(\)方法，来分析attachDetachController的主体处理逻辑。



主要调用了以下4个方法，下面会逐一分析：

（1）adc.populateActualStateOfWorld\(\)

（2）adc.populateDesiredStateOfWorld\(\)

（3）adc.reconciler.Run\(\)

（4）adc.desiredStateOfWorldPopulator.Run\(\)



**1 adc.populateActualStateOfWorld**

作用：初始化actualStateOfWorld结构体。



主要逻辑：遍历node对象，获取node.Status.VolumesAttached与node.Status.VolumesInUse，再调用adc.actualStateOfWorld.MarkVolumeAsAttached与adc.processVolumesInUse更新actualStateOfWorld信息。



**2 adc.populateDesiredStateOfWorld**

作用：初始化desiredStateOfWorld结构体。



主要逻辑：遍历pod列表，遍历pod的volume信息

（1）根据pod的volume信息来初始化desiredStateOfWorld；

（2）从actualStateOfWorld中查询，如果pod的volume已经attach到node了，则更新actualStateOfWorld，将该volume标记为已attach。



**3 adc.reconciler.Run**

主要是调用rc.reconcile做desiredStateOfWorld与actualStateOfWorld之间的调谐：对比desiredStateOfWorld与actualStateOfWorld，做attach与detach操作，更新actualStateOfWorld，并根据actualStateOfWorld更新node对象的.Status.VolumesAttached。



**3.1 rc.reconcile**

主要逻辑：

（1）遍历actualStateOfWorld中已经attached的volume，判断desiredStateOfWorld中是否存在，如果不存在，则调用rc.attacherDetacher.DetachVolume执行该volume的Detach操作；

（2）遍历desiredStateOfWorld中期望被attached的volume，判断actualStateOfWorld中是否已经attached到node上，如果没有，则先调用rc.isMultiAttachForbidden判断该volume的AccessModes是否支持多节点挂载，如支持，则继续调用rc.attacherDetacher.AttachVolume执行该volume的attach操作；

（3）调用rc.nodeStatusUpdater.UpdateNodeStatuses\(\)：根据从actualStateOfWorld获取已经attached到node的volume，更新node.Status.VolumesAttached的值。



###### rc.attachDesiredVolumes

主要逻辑：调用rc.attacherDetacher.AttachVolume触发attach逻辑。

###### rc.nodeStatusUpdater.UpdateNodeStatuses

主要逻辑：从actualStateOfWorld获取已经attach到node的volume，更新node对象的node.Status.VolumesAttached属性值。



**4 adc.desiredStateOfWorldPopulator.Run**

作用：更新desiredStateOfWorld，跟踪desiredStateOfWorld初始化后的后续变化更新。



主要调用了两个方法：

（1）dswp.findAndRemoveDeletedPods：更新desiredStateOfWorld，从中删除已经不存在的pod；

（2）dswp.findAndAddActivePods：更新desiredStateOfWorld，将新增的pod volume加入desiredStateOfWorld。



**4.1 dswp.findAndRemoveDeletedPods**

主要逻辑：

（1）从desiredStateOfWorld中取出pod列表；

（2）查询该pod对象是否还存在于etcd；

（3）不存在则调用dswp.desiredStateOfWorld.DeletePod将该pod从desiredStateOfWorld中删除。



#### 4.2 dswp.findAndAddActivePods\(\)

主要逻辑：  
（1）从etcd中获取全部pod信息；  
（2）调用util.ProcessPodVolumes，将新增pod volume加入desiredStateOfWorld。



**小结**

attachDetachController.Run中4个主要方法作用：

（1）adc.populateActualStateOfWorld\(\)：初始化actualStateOfWorld结构体。

（2）adc.populateDesiredStateOfWorld\(\)：初始化desiredStateOfWorld结构体。

（3）adc.reconciler.Run\(\)：做desiredStateOfWorld与actualStateOfWorld之间的调谐：对比desiredStateOfWorld与actualStateOfWorld，做attach与detach操作，更新actualStateOfWorld，并根据actualStateOfWorld更新node对象的.Status.VolumesAttached。

（4）adc.desiredStateOfWorldPopulator.Run\(\)：更新desiredStateOfWorld，跟踪desiredStateOfWorld初始化后的后续变化更新。





## NewAttachDetachController源码分析

接下来看到NewAttachDetachController方法，简单分析一下它的EventHandler。从代码中可以看到，主要是注册了pod对象与node对象的EventHandler。

```
// pkg/controller/volume/attachdetach/attach_detach_controller.go
// NewAttachDetachController returns a new instance of AttachDetachController.
func NewAttachDetachController(
...

podInformer.Informer().AddEventHandler(kcache.ResourceEventHandlerFuncs{
		AddFunc:    adc.podAdd,
		UpdateFunc: adc.podUpdate,
		DeleteFunc: adc.podDelete,
	})

...

nodeInformer.Informer().AddEventHandler(kcache.ResourceEventHandlerFuncs{
		AddFunc:    adc.nodeAdd,
		UpdateFunc: adc.nodeUpdate,
		DeleteFunc: adc.nodeDelete,
	})

...
}


```

#### 1 adc.podAdd

与adc.podUpdate一致，看下面adc.podUpdate的分析。

#### 2 adc.podUpdate

作用：主要是更新dsw，将新pod的volume加入dsw中。

主要看到util.ProcessPodVolumes方法。

###### util.ProcessPodVolumes

主要是更新dsw，将新pod的volume加入dsw中。

#### 3 adc.podDelete

作用：将volume从dsw中删除。

#### 4 adc.nodeAdd

主要是调用了adc.nodeUpdate方法进行处理，所以作用与adc.nodeUpdate基本相似，看adc.nodeUpdate的分析。

#### 5 adc.nodeUpdate

作用：往dsw中添加node，根据node.Status.VolumesInUse来更新asw。

###### 5.1 adc.addNodeToDswp

往dsw中添加node

###### 5.2 adc.processVolumesInUse

根据node.Status.VolumesInUse来更新asw

#### 6 adc.nodeDelete

作用：从dsw中删除node以及该node相关的挂载信息

小结

pod对象与node对象的EventHandler主要作用分别如下：

（1）adc.podAdd：更新dsw，将新pod的volume加入dsw中；

（2）adc.podUpdate：更新dsw，将新pod的volume加入dsw中；

（3）adc.podDelete：将volume从dsw中删除；

（4）adc.nodeAdd：往dsw中添加node，根据node.Status.VolumesInUse来更新asw；

（5）adc.nodeUpdate：往dsw中添加node，根据node.Status.VolumesInUse来更新asw；

（6）adc.nodeDelete：从dsw中删除node以及该node相关的挂载信息。



**总结**

**AD Cotroller作用**

AD Cotroller全称Attachment/Detachment 控制器，主要负责创建、删除VolumeAttachment对象，并调用volume plugin来做存储设备的Attach/Detach操作（将数据卷挂载到特定node节点上/从特定node节点上解除挂载），以及更新node.Status.VolumesAttached等。



不同的volume plugin的Attach/Detach操作逻辑有所不同，如通过ceph-csi（out-tree volume plugin）来使用ceph存储，则的Attach/Detach操作只是修改VolumeAttachment对象的状态，而不会真正的将数据卷挂载到节点/从节点上解除挂载。



**两个关键结构体**

（1）desiredStateOfWorld: 记录着集群中期望要挂载到node的pod的volume信息，简称DSW，数据来源：pod里声明的volume列表。



（2）actualStateOfWorld: 记录着集群中实际已经挂载到node节点的volume信息，简称ASW，数据来源：node对象中的node.Status.VolumesAttached属性值列表。



AD controller会做desiredStateOfWorld与actualStateOfWorld之间的调谐：对比desiredStateOfWorld与actualStateOfWorld，对volume做attach或detach操作。



