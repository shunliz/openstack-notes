# kubelet之volume manager源码分析 {#articleContentId}

基于tag v1.17.4

https://github.com/kubernetes/kubernetes/releases/tag/v1.17.4



**概述**

volume manager存在于kubelet中，主要是管理存储卷的attach/detach（与AD controller作用相同，通过kubelet启动参数控制哪个组件来做该操作，后续会详细介绍）、mount/umount等操作。



**简介**

容器的存储挂载分为两大步：

（1）attach；

（2）mount。



解除容器存储挂载分为两大步：

（1）umount；

（2）detach。



attach/detach操作可以由kube-controller-manager或者kubelet中的volume manager来完成，根据启动参数enable-controller-attach-detach来决定；而mount/umount操作只由kubelet中的volume manager来完成。



**VolumeManager接口**

（1）运行在kubelet 里让存储Ready的部件，主要是mount/unmount（attach/detach可选）；

（2）pod调度到这个node上后才会有卷的相应操作，所以它的触发端是kubelet（严格讲是kubelet里的pod manager），根据Pod Manager里pod spec里申明的存储来触发卷的挂载操作；

（3）Kubelet会监听到调度到该节点上的pod声明，会把pod缓存到Pod Manager中，VolumeManager通过Pod Manager获取PV/PVC的状态，并进行分析出具体的attach/detach、mount/umount, 操作然后调用plugin进行相应的业务处理。



**两个关键结构体**

（1）desiredStateOfWorld: 集群中期望要达到的数据卷挂载状态，简称DSW。假设集群内新调度了一个Pod，此时要用到volume，Pod被分配到某节点NodeA上。 此时，对于AD controller来说，DSW中节点NodeA应该有被分配的volume在准备被这个Pod挂载。



（2）actualStateOfWorld: 集群中实际存在的数据卷挂载状态，简称ASW。实际状态未必是和期望状态一样，比如实际状态Node上有刚调度过来的Pod，但是还没有相应已经attached状态的volume。



**actualStateOfWorld相关结构体**

**actualStateOfWorld**

实际存储挂载状态结构体。



actualStateOfWorld: 实际存储挂载状态，简称ASW。包括了已经成功挂载到node节点的存储，以及已经成功挂载该存储的pod列表。



主要属性attachedVolumes，数据结构map，key为已经成功挂载到node的存储名称，value为已经成功挂载到node节点的存储信息。

#### attachedVolume

主要属性mountedPods，数据结构map，key为pod名称，value为已经成功挂载了该存储的pod列表。

#### mountedPod

pod相关信息。



**desiredStateOfWorld相关结构体**

**desiredStateOfWorld**

期望存储挂载状态结构体。



desiredStateOfWorld: 期望的存储挂载状态，简称DSW。包括了期望挂载到node节点的存储，以及期望挂载该存储的pod列表。



主要属性volumesToMount，数据结构map，key为期望挂载到该node节点的存储，value为该存储相关信息。

#### volumeToMount

主要属性podsToMount，数据结构map，key为pod名称，value为期望挂载该存储的所有pod的相关信息。

#### podToMount

podToMount结构体主要记录了pod信息。



**方法入口分析**

kubelet管理volume的方式基于两个不同的状态：

（1）DesiredStateOfWorld：预期中，volume的挂载情况，简称预期状态。从pod对象中获取预期状态；

（2）ActualStateOfWorld：实际中，voluem的挂载情况，简称实际状态。从node.Status.VolumesAttached获取实际状态，并根据调谐更新实际状态。



Run方法中主要包含了2个方法：

（1）vm.desiredStateOfWorldPopulator.Run

（2）vm.reconciler.Run



下面将一一分析。

```
// pkg/kubelet/volumemanager/volume_manager.go
func (vm *volumeManager) Run(sourcesReady config.SourcesReady, stopCh <-chan struct{}) {
	defer runtime.HandleCrash()
    
    // 从apiserver同步pod信息，来更新DesiredStateOfWorld
	go vm.desiredStateOfWorldPopulator.Run(sourcesReady, stopCh)
	klog.V(2).Infof("The desired_state_of_world populator starts")

	klog.Infof("Starting Kubelet Volume Manager")
	// 预期状态和实际状态的协调者，负责调整实际状态至预期状态
	go vm.reconciler.Run(stopCh)

	metrics.Register(vm.actualStateOfWorld, vm.desiredStateOfWorld, vm.volumePluginMgr)

	if vm.kubeClient != nil {
		// start informer for CSIDriver
		vm.volumePluginMgr.Run(stopCh)
	}

	<-stopCh
	klog.Infof("Shutting down Kubelet Volume Manager")
}

```

**1 vm.desiredStateOfWorldPopulator.Run**

主要逻辑为调用dswp.populatorLoop\(\)，根据pod的volume信息，来更新DesiredStateOfWorld；并且不断循环调用该方法，来不断更新DesiredStateOfWorld。



dswp.loopSleepDuration的值为100ms。

**1.1 dswp.populatorLoop\(\)**

两个关键方法：

dswp.findAndAddNewPods\(\)：添加volume进DesiredStateOfWorld；

dswp.findAndRemoveDeletedPods\(\)：从DesiredStateOfWorld删除volume。

**1.1.1 dswp.findAndAddNewPods\(\)**

主要逻辑：

（1）如果kubelet开启了features.ExpandInUsePersistentVolumes，处理一下map mountedVolumesForPod，用于后续处理标记存储扩容逻辑；

（2）遍历pod列表，调用dswp.processPodVolumes将pod中的volume添加到DesiredStateOfWorld（pvc已经处于bound状态的volume才会添加到DesiredStateOfWorld）；同时processPodVolumes方法里有标记某存储是否需要扩容的逻辑，用于后续触发存储扩容操作。

dswp.processPodVolumes\(\)主要逻辑：

（1）调用dswp.podPreviouslyProcessed判断指定pod的volume是否已经被处理过了，处理过则直接返回；

（2）如果开启了features.ExpandInUsePersistentVolumes，则调用dswp.checkVolumeFSResize来标记需要扩容的volume信息；

（3）循环遍历pod.Spec.Volumes，做（4）（5）处理；

（4），调用dswp.createVolumeSpec，根据pvc与pv对象等信息，构造并返回volume.spec属性（方法中会获取pvc以及pv对象，并判断pvc对象是否与pv对象bound，没有bound则返回错误，另外，该方法还判断pvc的volumeMode等属性是否与pod内volume配置一致）；

（5）调用dswp.desiredStateOfWorld.AddPodToVolume将pod的volume信息加入desiredStateOfWorld中；

（6）调用dswp.markPodProcessed标记该pod的volume信息已被处理。





再来看一个关键方法checkVolumeFSResize\(\)：  


与存储扩容相关，当pv.Spec.Capacity大小大于pvc.Status.Capacity时，将该存储标记为需要扩容





**1.1.2 dswp.findAndRemoveDeletedPods**

主要逻辑：

（1）从dswp.desiredStateOfWorld中获取已挂载volume的pod，从podManager获取指定pod是否存在，不存在则再从containerRuntime中查询pod中的容器是否都已terminated，如以上两个条件都符合，则继续往下执行，否则直接返回；

（2）调用dswp.desiredStateOfWorld.DeletePodFromVolume，将pod从dsw.volumesToMount\[volumeName\].podsToMount\[podName\]中去除，即将volume挂载到指定pod的信息从desiredStateOfWorld中去除。

### 2 vm.reconciler.Run

主要是调用rc.reconcile\(\)，做存储的预期状态和实际状态的协调，负责调整实际状态至预期状态。

dswp.loopSleepDuration的值为100ms。



**2.1 rc.reconcile**

rc.reconcile\(\)主要逻辑：

（1）对于实际已经挂载了的存储，如果期望挂载信息中无该存储，或期望挂载存储的pod列表中没有该pod，则指定pod的指定存储需要unmount（最终调用csi.NodeUnpublishVolume）；

（2）从desiredStateOfWorld中获取需要mount到pod的volome信息列表：

a.当存储未attach到node时：调用方法将存储先attach到node上（此处会判断是不是由kubelet来做attach操作，是则创建VolumeAttachment对象并等待该对象的.status.attached的值为 true，不是则等待AD controller来做attach操作，kubelet将会根据node对象的.Status.VolumesAttached属性来判断该存储是否已attach到node上）；

b.当存储attach后，但未mount给pod或者需要remount时：调用方法进行volume mount（最终调用csi.NodeStageVolume与csi.NodePublishVolume）；

c.当存储需要扩容时，调用方法进行存储扩容（最终调用csi.NodeExpandVolume）；

（3）对比actualStateOfWorld，从desiredStateOfWorld中获取需要detached的volomes（detached意思为把存储从node上解除挂载）：

a.当actualStateOfWorld中表明，某volume没有被任何pod挂载，且desiredStateOfWorld中也不期望该volume被任何pod挂载，且attachedVolume.GloballyMounted属性为true时（device与global mount path的挂载关系还在），会调用到UnmountDevice，主要是调用csi.NodeUnstageVolume解除node上global mount path的存储挂载；

b.当actualStateOfWorld中表明，某volume没有被任何pod挂载，且desiredStateOfWorld中也不存在该volume，且attachedVolume.GloballyMounted属性为false时（已经调用过UnmountDevice，device与global mount path的挂载关系已解除），会调用到UnmountDevice，主要是从etcd中删除VolumeAttachment对象，并等待删除成功。



reconcile\(\)涉及主要方法：



（1）rc.operationExecutor.UnmountVolume：当actualStateOfWorld中表明，pod已经挂载了某volume，但desiredStateOfWorld中期望挂载某volume的pod列表中不存在该pod时（即表明存储已经挂载给pod，但该pod已经不存在了，需要解除该挂载），会调用到UnmountVolume，主要是调用csi.NodeUnpublishVolume将pod mount path解除挂载；



（2）rc.operationExecutor.AttachVolume：当actualStateOfWorld中已经挂载到node节点的volume信息中不存在某volume，但desiredStateOfWorld中期望某volume挂载到node节点上时（即表明需要挂载到node节点的存储未挂载），会调用到AttachVolume，主要是创建VolumeAttachment对象，并等待其.status.attached属性值更新为true；



（3）rc.operationExecutor.MountVolume：当desiredStateOfWorld中期望某volume挂载给某pod，但actualStateOfWorld中表明该volume并没有挂载给该pod，且该volume已经挂载到了node节点上，（或者该pod的volume需要remount），会调用到MountVolume，主要是调用csi.NodeStageVolume将存储挂载到node上的global mount path，调用csi.NodePublishVolume将存储从global mount path挂载到pod mount path；



（4）rc.operationExecutor.ExpandInUseVolume：主要负责在controller端的存储扩容操作完成后，做node端的存储扩容操作（后续会单独分析存储扩容操作）。



（5）rc.operationExecutor.UnmountDevice：当actualStateOfWorld中表明，某volume没有被任何pod挂载，且desiredStateOfWorld中也不期望该volume被任何pod挂载，且attachedVolume.GloballyMounted属性为true时（device与global mount path的挂载关系还在），会调用到UnmountDevice，主要是调用csi.NodeUnstageVolume解除node上global mount path的存储挂载；



（6）rc.operationExecutor.DetachVolume：当actualStateOfWorld中表明，某volume没有被任何pod挂载，且desiredStateOfWorld中也不存在该volume，且attachedVolume.GloballyMounted属性为false时（已经调用过UnmountDevice，device与global mount path的挂载关系已解除），会调用到UnmountDevice，主要是从etcd中删除VolumeAttachment对象，并等待删除成功。



pod挂载存储的调用流程：AttachVolume（csi-attacher.Attach） --&gt; MountVolume（csi-attacher.MountDevice --&gt; csi-mounter.SetUp）

解除pod存储挂载的调用流程：UnmountVolume（csi-mounter.TearDown） --&gt; UnmountDevice（csi-attacher.UnmountDevice） --&gt; DetachVolume（csi-attcher.Detach）

**2.1.1 rc.operationExecutor.VerifyControllerAttachedVolume**

因为attach/detach操作由AD controller来完成，所以volume manager只能通过node对象来获取指定volume是否已经attach，如已经attach，则更新actualStateOfWorld。



主要逻辑：从node对象中获取.Status.VolumesAttached，从而判断volume是否已经attach，然后更新actualStateOfWorld。



**2.2 rc.sync\(\)**

rc.sync\(\)调用时机：在vm.desiredStateOfWorldPopulator.Run中已经将所有pod的volume信息更新到了desiredStateOfWorld中。



rc.sync\(\)主要逻辑：扫描node上所有pod目录下的volume目录，来更新desiredStateOfWorld与actualStateOfWorld。



**总结**

**volume manager作用**

volume manager存在于kubelet中，主要是管理卷的attach/detach（与AD controller作用相同，通过kubelet启动参数控制哪个组件来做该操作）、mount/umount等操作。



**volume manager中pod挂载存储的调用流程**

AttachVolume（csi-attacher.Attach） --&gt; MountVolume（csi-attacher.MountDevice --&gt; csi-mounter.SetUp）



**volume manager中解除pod存储挂载的调用流程**

UnmountVolume（csi-mounter.TearDown） --&gt; UnmountDevice（csi-attacher.UnmountDevice） --&gt; DetachVolume（csi-attcher.Detach）



**volume manager的vm.reconciler.Run中各个方法的调用链**

（1）AttachVolume



Volume is not attached to node, kubelet attach is enabled



vm.reconciler.Run --&gt; rc.operationExecutor.AttachVolume --&gt; oe.operationGenerator.GenerateAttachVolumeFunc --&gt; csi-attacher.Attach（pkg/volume/csi/csi\_attacher.go）–&gt; create VolumeAttachment



（2）MountVolume



Volume is not mounted, or is already mounted, but requires remounting



vm.reconciler.Run --&gt; rc.operationExecutor.MountVolume --&gt; oe.operationGenerator.GenerateMountVolumeFunc --&gt; 1.csi-attacer.WaitForAttach（等待VolumeAttachment的.status.attached属性值更新为true） 2.csi-attacer.MountDevice（–&gt;csi.NodeStageVolume） 3.csi-mounter.SetUp（–&gt;csi.NodePublishVolume）



（3）ExpandInUseVolume



Volume is mounted， but it needs to resize



vm.reconciler.Run --&gt; rc.operationExecutor.ExpandInUseVolume --&gt; oe.operationGenerator.GenerateExpandInUseVolumeFunc --&gt; og.doOnlineExpansion --&gt; og.nodeExpandVolume --&gt; expander.NodeExpand \(pkg/volume/csi/expander.go\) --&gt; csi.NodeExpandVolume



（4）UnmountVolume



Volume is mounted, unmount it



vm.reconciler.Run --&gt; rc.operationExecutor.UnmountVolume --&gt; oe.operationGenerator.GenerateUnmountVolumeFunc --&gt; csi-mounter.TearDown（pkg/volume/csi/csi\_mounter.go）–&gt; csi.NodeUnpublishVolume



（5）UnmountDevice



Volume is globally mounted to device, unmount it



vm.reconciler.Run --&gt; rc.operationExecutor.UnmountDevice --&gt; oe.operationGenerator.GenerateUnmountVolumeFunc --&gt; csi-attacher.UnmountDevice --&gt; csi.NodeUnstageVolume



（6）DetachVolume



Volume is attached to node, detach it





