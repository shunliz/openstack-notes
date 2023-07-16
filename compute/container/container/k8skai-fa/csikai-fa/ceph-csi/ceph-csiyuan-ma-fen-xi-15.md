# kubelet分析-pvc扩容源码分析 {#articleContentId}

存储的扩容分为controller端操作与node端操作两大步骤，controller端操作由external-resizer来调用ceph完成，而node端操作由kubelet来完成，下面来分析下kubelet中有关存储扩容的相关代码。



基于tag v1.17.4

https://github.com/kubernetes/kubernetes/releases/tag/v1.17.4



**controller端存储扩容作用**

将底层存储扩容，如ceph rbd扩容，则会让ceph集群中的rbd image扩容。



controller端存储扩容分析请看external-resizer源码分析/pvc扩容分析



**node端存储扩容作用**

在pod所在的node上做相应的操作，让node感知该存储已经扩容，如ceph rbd filesystem扩容，则会调用node上的文件系统扩容命令让文件系统扩容。



某些存储无需进行node端扩容操作如cephfs。



**存储扩容大致过程**

（1）更改pvc.Spec.Resources.Requests.storgage，触发扩容



（2）controller端存储扩容：external-resizer watch pvc对象，当发现pvc.Spec.Resources.Requests.storgage比pvc.Status.Capacity.storgage大，于是调csi plugin的ControllerExpandVolume方法进行 controller端扩容，进行底层存储扩容，并更新pv.Spec.Capacity.storgage。



（3）node端存储扩容：kubelet发现pv.Spec.Capacity.storage大于pvc.Status.Capacity.storage，于是调csi node端扩容，对dnode上文件系统扩容，成功后kubelet更新pvc.Status.Capacity.storage。



**存储扩容详细流程**

下面以ceph rbd存储扩容为例，对详细的存储扩容过程进行分析。

![](/assets/compute-container-k8s-cephcsi13633151.png)1）修改pvc对象，修改申请存储大小（pvc.spec.resources.requests.storage）；



（2）修改成功后，external-resizer监听到该pvc的update事件，发现pvc.Spec.Resources.Requests.storgage比pvc.Status.Capacity.storgage大，于是调ceph-csi组件进行 controller端扩容；



（3）ceph-csi组件调用ceph存储，进行底层存储扩容；



（4）底层存储扩容完成后，external-resizer组件更新pv对象的.Spec.Capacity.storgage的值为扩容后的存储大小；



（5）kubelet的volume manager在reconcile\(\)调谐过程中发现pv.Spec.Capacity.storage大于pvc.Status.Capacity.storage，于是调ceph-csi组件进行 node端扩容；



（6）ceph-csi组件对node上存储对应的文件系统扩容；



（7）扩容完成后，kubelet更新pvc.Status.Capacity.storage的值为扩容后的存储大小。



下面主要对kubelet中的存储扩容相关的代码进行分析，controller端存储扩容分析将在后续分析external-resizer时进行分析。



volumeManager.Run

关于存储扩容，主要看到两个主要方法：

（1）vm.desiredStateOfWorldPopulator.Run：主要负责找到并标记需要扩容的存储；

（2）vm.reconciler.Run：主要负责对需要扩容的存储触发进行扩容操作。

## 1.vm.desiredStateOfWorldPopulator.Run

主要是对dswp.populatorLoop的调用

```
func (dswp *desiredStateOfWorldPopulator) Run(sourcesReady config.SourcesReady, stopCh <-chan struct{}) {
	// Wait for the completion of a loop that started after sources are all ready, then set hasAddedPods accordingly
	klog.Infof("Desired state populator starts to run")
	wait.PollUntil(dswp.loopSleepDuration, func() (bool, error) {
		done := sourcesReady.AllReady()
		dswp.populatorLoop()
		return done, nil
	}, stopCh)
	dswp.hasAddedPodsLock.Lock()
	dswp.hasAddedPods = true
	dswp.hasAddedPodsLock.Unlock()
	wait.Until(dswp.populatorLoop, dswp.loopSleepDuration, stopCh)
}

```

populatorLoop中调用dswp.findAndAddNewPods

```
func (dswp *desiredStateOfWorldPopulator) populatorLoop() {
	dswp.findAndAddNewPods()

	// findAndRemoveDeletedPods() calls out to the container runtime to
	// determine if the containers for a given pod are terminated. This is
	// an expensive operation, therefore we limit the rate that
	// findAndRemoveDeletedPods() is called independently of the main
	// populator loop.
	if time.Since(dswp.timeOfLastGetPodStatus) < dswp.getPodStatusRetryDuration {
		klog.V(5).Infof(
			"Skipping findAndRemoveDeletedPods(). Not permitted until %v (getPodStatusRetryDuration %v).",
			dswp.timeOfLastGetPodStatus.Add(dswp.getPodStatusRetryDuration),
			dswp.getPodStatusRetryDuration)

		return
	}

	dswp.findAndRemoveDeletedPods()
}

```

findAndAddNewPods中主要看到dswp.processPodVolumes

```
// Iterate through all pods and add to desired state of world if they don't
// exist but should
func (dswp *desiredStateOfWorldPopulator) findAndAddNewPods() {
	// Map unique pod name to outer volume name to MountedVolume.
	mountedVolumesForPod := make(map[volumetypes.UniquePodName]map[string]cache.MountedVolume)
	if utilfeature.DefaultFeatureGate.Enabled(features.ExpandInUsePersistentVolumes) {
		for _, mountedVolume := range dswp.actualStateOfWorld.GetMountedVolumes() {
			mountedVolumes, exist := mountedVolumesForPod[mountedVolume.PodName]
			if !exist {
				mountedVolumes = make(map[string]cache.MountedVolume)
				mountedVolumesForPod[mountedVolume.PodName] = mountedVolumes
			}
			mountedVolumes[mountedVolume.OuterVolumeSpecName] = mountedVolume
		}
	}

	processedVolumesForFSResize := sets.NewString()
	for _, pod := range dswp.podManager.GetPods() {
		if dswp.isPodTerminated(pod) {
			// Do not (re)add volumes for terminated pods
			continue
		}
		dswp.processPodVolumes(pod, mountedVolumesForPod, processedVolumesForFSResize)
	}
}

```

processPodVolumes主要是调用dswp.checkVolumeFSResize对需要扩容的存储进行标记

```
// processPodVolumes processes the volumes in the given pod and adds them to the
// desired state of the world.
func (dswp *desiredStateOfWorldPopulator) processPodVolumes(
	pod *v1.Pod,
	mountedVolumesForPod map[volumetypes.UniquePodName]map[string]cache.MountedVolume,
	processedVolumesForFSResize sets.String) {
	
	......

	expandInUsePV := utilfeature.DefaultFeatureGate.Enabled(features.ExpandInUsePersistentVolumes)
	// Process volume spec for each volume defined in pod
	for _, podVolume := range pod.Spec.Volumes {
		
		......

		if expandInUsePV {
			dswp.checkVolumeFSResize(pod, podVolume, pvc, volumeSpec,
				uniquePodName, mountedVolumesForPod, processedVolumesForFSResize)
		}
	}

	......

}

```

#### 1.1 checkVolumeFSResize

主要逻辑：  
（1）调用volumeRequiresFSResize判断是否需要扩容；  
（2）调用dswp.actualStateOfWorld.MarkFSResizeRequired做进标记处理。

#### 1.1.1 volumeRequiresFSResize

pv.Spec.Capacity.storage大小比pvc.Status.Capacity.storage大小要大时返回true

#### **1.1.2 MarkFSResizeRequired**

主要逻辑：

（1）获取volume对应的volumePlugin；

（2）调用volumePlugin.RequiresFSResize\(\)判断plugin是否支持resize；

（3）plugin支持则设置podObj的fsResizeRequired属性为true。（reconcile中会根据podObj的fsResizeRequired属性为true来触发node端resize操作）

## 2.vm.reconciler.Run

```
func (rc *reconciler) Run(stopCh <-chan struct{}) {
	wait.Until(rc.reconciliationLoopFunc(), rc.loopSleepDuration, stopCh)
}

func (rc *reconciler) reconciliationLoopFunc() func() {
	return func() {
		rc.reconcile()

		// Sync the state with the reality once after all existing pods are added to the desired state from all sources.
		// Otherwise, the reconstruct process may clean up pods' volumes that are still in use because
		// desired state of world does not contain a complete list of pods.
		if rc.populatorHasAddedPods() && !rc.StatesHasBeenSynced() {
			klog.Infof("Reconciler: start to sync state")
			rc.sync()
		}
	}
}

```

省略了部分代码，下面列出的是扩容相关代码。



扩容相关主要逻辑：

（1）调用rc.actualStateOfWorld.PodExistsInVolume；

（2）判断上一步骤的返回是否是IsFSResizeRequiredError，true时调用rc.operationExecutor.ExpandInUseVolume触发扩容操作。

**2.1 rc.actualStateOfWorld.PodExistsInVolume**

扩容相关主要逻辑：

（1）从actualStateOfWorld中获取获取volumeObj；

（2）从volumeObj中获取podObj；

（3）判断podObj的fsResizeRequired属性，true时返回newFsResizeRequiredError。

#### 2.2 rc.operationExecutor.ExpandInUseVolume

调用oe.operationGenerator.GenerateExpandInUseVolumeFunc做进一步处理

GenerateExpandInUseVolumeFunc中主要看到og.doOnlineExpansion

###### og.doOnlineExpansion

doOnlineExpansion主要是调用og.nodeExpandVolume

**og.nodeExpandVolume**

og.nodeExpandVolume主要逻辑：

（1）获取扩容plugin；

（2）获取pv与pvc对象；

（3）当pv.Spec.Capacity比pvc.Status.Capacity大时，调用expandableVolumePlugin.NodeExpand进行扩容；

（4）扩容完成，调用util.MarkFSResizeFinished，更新PVC.Status.Capacity.storage的值为扩容后的存储大小值。



###### expandableVolumePlugin.NodeExpand

NodeExpand中会调用util.CheckVolumeModeFilesystem来检查volumemode是否是block，如果是block，则不用进行node端扩容操作。

###### MarkFSResizeFinished

更新PVC对象，将.Status.Capacity.storage的值为扩容后的存储大小值



**总结**

存储的扩容分为controller端操作与node端操作两大步骤，controller端操作由external-resizer来调用ceph完成，而node端操作由kubelet来完成。



**controller端存储扩容作用**

将底层存储扩容，如ceph rbd扩容，则会让ceph集群中的rbd image扩容。



**node端存储扩容作用**

在pod所在的node上做相应的操作，让node感知该存储已经扩容，如ceph rbd filesystem扩容，则会调用node上的文件系统扩容命令让文件系统扩容。



某些存储无需进行node端扩容操作如cephfs。



**存储扩容大致过程**

（1）更改pvc.Spec.Resources.Requests.storgage，触发扩容



（2）controller端存储扩容：external-resizer watch pvc对象，当发现pvc.Spec.Resources.Requests.storgage比pvc.Status.Capacity.storgage大，于是调csi plugin的ControllerExpandVolume方法进行 controller端扩容，进行底层存储扩容，并更新pv.Spec.Capacity.storgage。



（3）node端存储扩容：kubelet发现pv.Spec.Capacity.storage大于pvc.Status.Capacity.storage，于是调csi node端扩容，对dnode上文件系统扩容，成功后kubelet更新pvc.Status.Capacity.storage。



**存储扩容整体流程**

整体的存储扩容步骤如下：

（1）修改pvc对象，修改申请存储大小（pvc.spec.resources.requests.storage）；



（2）修改成功后，external-resizer监听到该pvc的update事件，发现pvc.Spec.Resources.Requests.storgage比pvc.Status.Capacity.storgage大，于是调ceph-csi组件进行 controller端扩容；



（3）ceph-csi组件调用ceph存储，进行底层存储扩容；



（4）底层存储扩容完成后，ceph-csi组件更新pv对象的.Spec.Capacity.storgage的值为扩容后的存储大小；



（5）kubelet的volume manager在reconcile\(\)调谐过程中发现pv.Spec.Capacity.storage大于pvc.Status.Capacity.storage，于是调ceph-csi组件进行 node端扩容；



（6）ceph-csi组件对node上存储对应的文件系统扩容；



（7）扩容完成后，kubelet更新pvc.Status.Capacity.storage的值为扩容后的存储大小。



node端（kubelet）存储扩容调用链

vm.reconciler.Run --&gt; rc.operationExecutor.ExpandInUseVolume --&gt; oe.operationGenerator.GenerateExpandInUseVolumeFunc --&gt; og.doOnlineExpansion --&gt; og.nodeExpandVolume --&gt; expander.NodeExpand \(pkg/volume/csi/expander.go\) --&gt; csClient.NodeExpandVolume



