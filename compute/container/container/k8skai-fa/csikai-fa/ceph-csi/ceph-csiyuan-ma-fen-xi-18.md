# external-attacher源码分析-核心处理逻辑分析 {#articleContentId}

external-attacher

external-attacher属于external plugin中的一个。下面我们先来回顾一下external plugin以及csi系统结构。



external plugin

external plugin包括了external-provisioner、external-attacher、external-resizer、external-snapshotter等，external plugin辅助csi plugin组件，共同完成了存储相关操作。external plugin负责watch pvc、volumeAttachment等对象，然后调用volume plugin来完成存储的相关操作。如external-provisioner watch pvc对象，然后调用csi plugin来创建存储，最后创建pv对象；external-attacher watch volumeAttachment对象，然后调用csi plugin来做attach/dettach操作，并修改volumeAttachment对象与pv对象；external-resizer watch pvc对象，然后调用csi plugin来做存储的扩容操作等。

![](/assets/compute-container-k8s-cephcsi12633181.png)**external-attacher作用分析**

根据CSI plugin是否支持ControllerPublish/ControllerUnpublish操作，external-attacher的作用分为如下两种：

（1）当CSI plugin不支持ControllerPublish/ControllerUnpublish操作时，AD controller（或kubelet的volume manager）创建VolumeAttachment对象后，external-attacher仅参与VolumeAttachment对象的修改，将attached属性值patch为true；而external-attacher对pv对象无任何同步处理操作。

（2）当CSI plugin支持ControllerPublish/ControllerUnpublish操作时，external-attacher调用csi plugin（ControllerPublishVolume）进行存储的attach操作，然后更改VolumeAttachment对象，将attached属性值patch为true，并patch pv对象，增加该external-attacher相关的finalizer；对于pv对象，external-attacher负责处理pv对象的finalizer，patch pv对象，去除该external-attacher相关的finalizer（该external-attacher执行attach操作时添加的finalizer）。



**源码分析**

external-attacher的源码分析将分为两部分：

（1）main方法以及启动参数分析；

（2）核心处理逻辑分析。



**Run**

核心逻辑为跑goroutine不停的调用ctrl.syncVA与ctrl.syncPV对VolumeAttachment对象以及PV对象进行同步处理。

```
//external-attcher/pkg/controller/controller.go

// Run starts CSI attacher and listens on channel events
func (ctrl *CSIAttachController) Run(workers int, stopCh <-chan struct{}) {
	defer ctrl.vaQueue.ShutDown()
	defer ctrl.pvQueue.ShutDown()

	klog.Infof("Starting CSI attacher")
	defer klog.Infof("Shutting CSI attacher")

	if !cache.WaitForCacheSync(stopCh, ctrl.vaListerSynced, ctrl.pvListerSynced) {
		klog.Errorf("Cannot sync caches")
		return
	}
	for i := 0; i < workers; i++ {
		go wait.Until(ctrl.syncVA, 0, stopCh)
		go wait.Until(ctrl.syncPV, 0, stopCh)
	}

	if ctrl.shouldReconcileVolumeAttachment {
		go wait.Until(func() {
			err := ctrl.handler.ReconcileVA()
			if err != nil {
				klog.Errorf("Failed to reconcile volume attachments: %v", err)
			}
		}, ctrl.reconcileSync, stopCh)
	}

	<-stopCh
}

```

**1.syncVA**

syncVA负责VolumeAttachment对象的处理，核心逻辑为：

（1）判断VolumeAttachment对象的.Spec.Attacher属性值，判断是否由本attacher组件来负责VolumeAttachment对象的同步处理；

（2）调用ctrl.handler.SyncNewOrUpdatedVolumeAttachment\(va\)。

**1.1 SyncNewOrUpdatedVolumeAttachment**

ctrl.handler.SyncNewOrUpdatedVolumeAttachment\(\)包含两个实现，将根据CSI plugin是否支持ControllerPublish/ControllerUnpublish操作来调用不同的实现。（调用不同实现的判断逻辑在external-attacher的main方法里，前面分析external-attacher的main方法时已经分析过了，忘记的可以回去看下）



（1）trivialHandler.SyncNewOrUpdatedVolumeAttachment：当CSI plugin不支持ControllerPublish/ControllerUnpublish操作时，AD controller（或kubelet的volume manager）创建VolumeAttachment对象后，external-attacher仅参与VolumeAttachment对象的修改，将attached属性值patch为true。

（2）csiHandler.SyncNewOrUpdatedVolumeAttachment：当CSI plugin支持ControllerPublish/ControllerUnpublish操作时，external-attacher调用csi plugin（ControllerPublishVolume）进行存储的attach操作，然后更改VolumeAttachment对象，将attached属性值patch为true。



**1.1.1 trivialHandler.SyncNewOrUpdatedVolumeAttachment**

先来看trivialHandler的实现。



当VolumeAttachment的attached属性值为false时，调用markAsAttached做进一步处理。

###### markAsAttached

markAsAttached主要是将VolumeAttachment的attached属性值patch为true。

**1.1.2 csiHandler.SyncNewOrUpdatedVolumeAttachment**

先来看csiHandler的实现。主要逻辑如下：

（1）当VolumeAttachment对象的.DeletionTimestamp字段为空，调用h.syncAttach（调用了csi plugin的ControllerPublishVolume方法来做存储的attach操作，并调用markAsAttached将VolumeAttachment的attached属性值patch为true）；

（2）当VolumeAttachment对象的.DeletionTimestamp字段不为空，调用h.syncDetach（调用了csi plugin的ControllerUnpublishVolume方法来做存储的dettach操作，并调用markAsDetached将VolumeAttachment的attached属性值patch为false）。

**syncAttach**

syncAttach不展开分析了，感兴趣的可以自己深入分析，从h.csiAttach调用入手。



主要逻辑为：

（1）调用h.csiAttach做attach操作（patch pv对象，增加该external-attacher相关的Finalizer，并最终调用了csi plugin的ControllerPublishVolume方法来做存储的attach操作）；

（2）调用markAsAttached将VolumeAttachment的attached属性值patch为true。

**syncDetach**

同样的，syncDetach不展开分析，这里直接给出结果，最终调用了csi plugin的ControllerUnpublishVolume方法来做存储的dettach操作，并调用markAsDetached将VolumeAttachment的attached属性值patch为false，感兴趣的可以自己深入分析，从h.csiDetach调用入手。

#### 2.syncPV

syncPV负责PV对象的处理，核心逻辑为调用ctrl.handler.SyncNewOrUpdatedPersistentVolume\(pv\)来对pv对象做进一步处理。

**2.1 SyncNewOrUpdatedPersistentVolume**

跟上面分析的syncVA中的SyncNewOrUpdatedVolumeAttachment一样，ctrl.handler.SyncNewOrUpdatedPersistentVolume\(\)也包含两个实现，将根据CSI plugin是否支持ControllerPublish/ControllerUnpublish操作来调用不同的实现。（调用不同实现的判断逻辑在external-attacher的main方法里，前面分析external-attacher的main方法时已经分析过了，忘记的可以回去看下）



（1）trivialHandler.SyncNewOrUpdatedPersistentVolume：当CSI plugin不支持ControllerPublish/ControllerUnpublish操作时，external-attacher不对pv对象做任何同步处理操做。

（2）csiHandler.SyncNewOrUpdatedPersistentVolume：当CSI plugin支持ControllerPublish/ControllerUnpublish操作时，external-attacher负责处理pv对象的finalizer，patch pv对象，去除该external-attacher相关的finalizer（该external-attacher执行attach操作时添加的finalizer）。



**2.1.1 trivialHandler.SyncNewOrUpdatedPersistentVolume**

trivialHandler.SyncNewOrUpdatedPersistentVolume方法直接返回，可以看出对于pv对象，不做处理。





**2.1.2 csiHandler.SyncNewOrUpdatedPersistentVolume**

csiHandler.SyncNewOrUpdatedPersistentVolume主要是处理pv对象的finalizer，patch pv对象，去除该external-attacher相关的finalizer（该external-attacher执行attach操作时添加的finalizer）。



主要逻辑：

（1）判断pv对象的DeletionTimestamp是否为空，为空则直接返回；

（2）检查pv对象是否含有该external-attacher相关的finalizer，没有则直接返回；

（3）查询volumeAttachment对象列表，遍历查询是否有va对象记录着该pv，有则直接返回；

（4）去除pv对象中该external-attacher相关的finalizer；

（5）patch pv对象。



**总结**

external-attacher属于external plugin中的一个。



**external-attacher作用分析**

根据CSI plugin是否支持ControllerPublish/ControllerUnpublish操作，external-attacher的作用分为如下两种：

（1）当CSI plugin不支持ControllerPublish/ControllerUnpublish操作时，AD controller（或kubelet的volume manager）创建VolumeAttachment对象后，external-attacher仅参与VolumeAttachment对象的修改，将attached属性值patch为true；而external-attacher对pv对象无任何同步处理操作。

（2）当CSI plugin支持ControllerPublish/ControllerUnpublish操作时，external-attacher调用csi plugin（ControllerPublishVolume）进行存储的attach操作，然后更改VolumeAttachment对象，将attached属性值patch为true，并patch pv对象，增加该external-attacher相关的Finalizer；对于pv对象，external-attacher负责处理pv对象的finalizer，patch pv对象，去除该external-attacher相关的finalizer（该external-attacher执行attach操作时添加的finalizer）。



**external-attacher与ceph-csi rbd结合使用**

ceph-csi不支持ControllerPublish/ControllerUnpublish操作，所以external-attacher与ceph-csi rbd结合使用，external-attacher仅参与VolumeAttachment对象的修改，将attached属性值patch为true。



