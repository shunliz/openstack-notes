# kube-controller-manager之PV Cotroller源码分析 {#articleContentId}



**概述**

kube-controller-manager组件中，有两个controller与存储相关，分别是PV controller与AD controller。



基于tag v1.17.4

https://github.com/kubernetes/kubernetes/releases/tag/v1.17.4



**PV Cotroller分析**

这节先对PV controller进行分析。



涉及主要对象

（1）Persistent Volume \(PV\)： 持久化存储卷，详细定义了存储的各项参数。

（2）Persistent Volume Claim \(PVC\)：持久化存储卷的使用声明，也即说明需要什么样的多大的存储。

（3）StorageClass：创建pv的模板，定义了创建存储的模板参数。



**PV对象主要状态变更**

（1）available --&gt; bound：一个pv对象创建出来后，处于available状态。pv controller会为pvc对象寻找合适的pv对象与之绑定，随即pv对象状态变更为bound。

（2）bound --&gt; released：当与pv绑定的pvc对象被删除后，如果回收逻辑为retain，则pv对象状态变更为released。



**pvc对象主要状态变更**

（1）pending --&gt; bound：一个pvc对象创建出来后，处于pending状态。pv controller会为pvc对象寻找合适的pv对象与之绑定，随即pvc对象状态变更为bound。



**pvc如何选择合适的pv来绑定？**

（1）volumeName匹配：当pvc对象中指定了volumeName属性，则会直接查询名称为该volumeName属性值一致的pv，并与之绑定，当该pv不存在时，该pvc会一直处于pending状态；

（2）volumeMode匹配：选择具有与pvc相同的volumeMode的pv（Block/FileSystem）；

（3）storageclass匹配：选择具有与pvc相同的stroageclass名称的pv；

（4）accessMode匹配：选择具有与pvc相同的accessMode的pv；

（5）size检查：选择size大于等于且最接近pvc的size声明的pv。



其他：当一个 PVC 找不到合适的 PV 时，相应的volume plugin就会根据 StorageClass对象的参数配置去做一个动态创建 PV 的操作；而当存在一个合适的 PV 时，就会直接与现有的 PV绑定，而不再去动态创建。当pvc的volumeName属性不为空时，任何情况下都不会触发动态创建pv的操作。



**pv与pvc提前绑定特性**

当一个pv的spec.claimRef属性指定了pvc时，则该pv只会与指定pvc绑定，不会与其他pvc绑定。示例如下：

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pvc-226aad72-c9ca-48d7-a0b2-c0f7599a3132
spec:
  claimRef:
    apiVersion: v1
    kind: PersistentVolumeClaim
    name: rbd-test-1
    namespace: test
    resourceVersion: "260347739"
    uid: 226aad72-c9ca-48d7-a0b2-c0f7599a3132
...

```

由out-tree volume plugin来创建的pv，就使用了pv与pvc提前绑定的特性，创建pv时带上spec.claimRef属性，指定了特定的pvc，防止并发操作时出现多个pvc与多个pv交叉绑定导致出错的情况。



**PV Cotroller作用**

PV Cotroller全称PersistentVolume controller，主要负责：

（1）pv、pvc对象的绑定；

（2）pv、pvc对象的生命周期管理（如创建/删除底层存储，创建/删除pv对象，pv与pvc对象的状态变更）。



**注意**

（1）当一个pvc创建出来后，pv controller会先寻找现存的合适的pv与之绑定，当找不到合适的pv时，才会去创建新的pv。



（2）前面说过，根据源码所在位置，volume plugin分为in-tree与out-tree两个部分。



（3）创建/删除底层存储、创建/删除pv对象的操作，由PV controller调用volume plugin（in-tree）来完成。如果是k8s通过ceph-csi（csi plugin）来使用ceph存储，volume plugin为ceph-csi，属于out-tree，所以创建/删除底层存储、创建/删除pv对象的操作由external-provisioner来完成。



PV和PVC的源码处理逻辑都在kubernetes/pkg/controller/volume/persistentvolume/pv\_controller\_base.go和kubernetes/pkg/controller/volume/persistentvolume/pv\_controller.go这两个文件中。



**源码分析入口**

直接看到PersistentVolumeController的Run方法，主要就是起了三个Goroutine，分别运行3个方法，下面将一一分析。

```
// kubernetes/pkg/controller/volume/persistentvolume/pv_controller_base.go

func (ctrl *PersistentVolumeController) Run(stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()
	defer ctrl.claimQueue.ShutDown()
	defer ctrl.volumeQueue.ShutDown()

	klog.Infof("Starting persistent volume controller")
	defer klog.Infof("Shutting down persistent volume controller")

	if !cache.WaitForNamedCacheSync("persistent volume", stopCh, ctrl.volumeListerSynced, ctrl.claimListerSynced, ctrl.classListerSynced, ctrl.podListerSynced, ctrl.NodeListerSynced) {
		return
	}

	ctrl.initializeCaches(ctrl.volumeLister, ctrl.claimLister)

	go wait.Until(ctrl.resync, ctrl.resyncPeriod, stopCh)
	go wait.Until(ctrl.volumeWorker, time.Second, stopCh)
	go wait.Until(ctrl.claimWorker, time.Second, stopCh)

	metrics.Register(ctrl.volumes.store, ctrl.claims)

	<-stopCh
}

```

**1 ctrl.resync**

resync方法十分简单，主要作用是定时循环查询出pv和pvc列表，然后放入到队列volumeQueue和claimQueue中，让volumeWorker和claimWorker进行消费。



**2 ctrl.volumeWorker**

volumeWorker方法主要是维护pv的状态，根据不同的状况对pv对象的状态值进行更新。当找不到与pv绑定的pvc时，会调用volume plugin来删除底层存储，并删除pv对象（这里包含了in-tree与out-tree两条逻辑）。



volumeWorker会不断循环消费volumeQueue队列里面的数据，然后获取到相应的PV执行updateVolume操作。

```
//kubernetes/pkg/controller/volume/persistentvolume/pv_controller_base.go

func (ctrl *PersistentVolumeController) volumeWorker() {
	workFunc := func() bool {
		keyObj, quit := ctrl.volumeQueue.Get()
		if quit {
			return true
		}
		defer ctrl.volumeQueue.Done(keyObj)
		key := keyObj.(string)
		klog.V(5).Infof("volumeWorker[%s]", key)

		_, name, err := cache.SplitMetaNamespaceKey(key)
		if err != nil {
			klog.V(4).Infof("error getting name of volume %q to get volume from informer: %v", key, err)
			return false
		}
		volume, err := ctrl.volumeLister.Get(name)
		if err == nil {
			// The volume still exists in informer cache, the event must have
			// been add/update/sync
			ctrl.updateVolume(volume)
			return false
		}
		if !errors.IsNotFound(err) {
			klog.V(2).Infof("error getting volume %q from informer: %v", key, err)
			return false
		}

		// The volume is not in informer cache, the event must have been
		// "delete"
		volumeObj, found, err := ctrl.volumes.store.GetByKey(key)
		if err != nil {
			klog.V(2).Infof("error getting volume %q from cache: %v", key, err)
			return false
		}
		if !found {
			// The controller has already processed the delete event and
			// deleted the volume from its cache
			klog.V(2).Infof("deletion of volume %q was already processed", key)
			return false
		}
		volume, ok := volumeObj.(*v1.PersistentVolume)
		if !ok {
			klog.Errorf("expected volume, got %+v", volumeObj)
			return false
		}
		ctrl.deleteVolume(volume)
		return false
	}
	for {
		if quit := workFunc(); quit {
			klog.Infof("volume worker queue shutting down")
			return
		}
	}
}

```

#### 2.1 ctrl.updateVolume

updateVolume方法会调用syncVolume方法，执行核心流程。

```
func (ctrl *PersistentVolumeController) updateVolume(volume *v1.PersistentVolume) {
    // Store the new volume version in the cache and do not process it if this
    // is an old version.
    //更新缓存
    new, err := ctrl.storeVolumeUpdate(volume)
    if err != nil {
        klog.Errorf("%v", err)
    }
    if !new {
        return
    }
    //核心方法，根据当前 PV 对象的规格对 PV 和 PVC 进行绑定或者解绑
    err = ctrl.syncVolume(volume)
    if err != nil {
        if errors.IsConflict(err) {
            // Version conflict error happens quite often and the controller
            // recovers from it easily.
            klog.V(3).Infof("could not sync volume %q: %+v", volume.Name, err)
        } else {
            klog.Errorf("could not sync volume %q: %+v", volume.Name, err)
        }
    }
}

```

**ctrl.syncVolume**

syncVolume方法为核心方法，主要调谐更新pv的状态:

（1）如果spec.claimRef未设置，则是未使用过的pv，则调用updateVolumePhase函数更新状态设置 phase 为 available；

（2）如果spec.claimRef不为空，则该pv已经与pvc bound过了，此时若对应的pvc不存在，则更新pv状态为released；

（3）如果pv对应的pvc被删除了，调用ctrl.reclaimVolume根据pv的回收策略进行相应操作，如果是retain，则不做操作，如果是delete，则调用volume plugin来删除底层存储，并删除pv对象（当volume plugin为csi时，将走out-tree逻辑，pv controller不做删除存储与pv对象的操作，由external provisioner组件来完成该操作）。

```
func (ctrl *PersistentVolumeController) syncVolume(volume *v1.PersistentVolume) error {
    klog.V(4).Infof("synchronizing PersistentVolume[%s]: %s", volume.Name, getVolumeStatusForLogging(volume)) 
    ...
    //如果spec.claimRef未设置，则是未使用过的pv，则调用updateVolumePhase函数更新状态设置 phase 为 available
    if volume.Spec.ClaimRef == nil { 
        klog.V(4).Infof("synchronizing PersistentVolume[%s]: volume is unused", volume.Name)
        if _, err := ctrl.updateVolumePhase(volume, v1.VolumeAvailable, ""); err != nil { 
            return err
        }
        return nil
    } else /* pv.Spec.ClaimRef != nil */ { 
        //正在被bound中，更新状态available
        if volume.Spec.ClaimRef.UID == "" { 
            klog.V(4).Infof("synchronizing PersistentVolume[%s]: volume is pre-bound to claim %s", volume.Name, claimrefToClaimKey(volume.Spec.ClaimRef))
            if _, err := ctrl.updateVolumePhase(volume, v1.VolumeAvailable, ""); err != nil { 
                return err
            }
            return nil
        }
        klog.V(4).Infof("synchronizing PersistentVolume[%s]: volume is bound to claim %s", volume.Name, claimrefToClaimKey(volume.Spec.ClaimRef))
        // Get the PVC by _name_
        var claim *v1.PersistentVolumeClaim
        //根据 pv 的 claimRef 获得 pvc
        claimName := claimrefToClaimKey(volume.Spec.ClaimRef)
        obj, found, err := ctrl.claims.GetByKey(claimName)
        if err != nil {
            return err
        }
        //如果在队列未发现，可能是volume被删除了，或者失败了，重新同步pvc
        if !found && metav1.HasAnnotation(volume.ObjectMeta, pvutil.AnnBoundByController) { 
            if volume.Status.Phase != v1.VolumeReleased && volume.Status.Phase != v1.VolumeFailed {
                obj, err = ctrl.claimLister.PersistentVolumeClaims(volume.Spec.ClaimRef.Namespace).Get(volume.Spec.ClaimRef.Name)
                if err != nil && !apierrors.IsNotFound(err) {
                    return err
                }
                found = !apierrors.IsNotFound(err)
                if !found {
                    obj, err = ctrl.kubeClient.CoreV1().PersistentVolumeClaims(volume.Spec.ClaimRef.Namespace).Get(context.TODO(), volume.Spec.ClaimRef.Name, metav1.GetOptions{})
                    if err != nil && !apierrors.IsNotFound(err) {
                        return err
                    }
                    found = !apierrors.IsNotFound(err)
                }
            }
        }
        if !found {
            klog.V(4).Infof("synchronizing PersistentVolume[%s]: claim %s not found", volume.Name, claimrefToClaimKey(volume.Spec.ClaimRef)) 
        } else {
            var ok bool
            claim, ok = obj.(*v1.PersistentVolumeClaim)
            if !ok {
                return fmt.Errorf("Cannot convert object from volume cache to volume %q!?: %#v", claim.Spec.VolumeName, obj)
            }
            klog.V(4).Infof("synchronizing PersistentVolume[%s]: claim %s found: %s", volume.Name, claimrefToClaimKey(volume.Spec.ClaimRef), getClaimStatusForLogging(claim))
        }
        if claim != nil && claim.UID != volume.Spec.ClaimRef.UID { 
            klog.V(4).Infof("synchronizing PersistentVolume[%s]: claim %s has different UID, the old one must have been deleted", volume.Name, claimrefToClaimKey(volume.Spec.ClaimRef))
            // Treat the volume as bound to a missing claim.
            claim = nil
        }
        //claim可能被删除了，或者pv被删除了
        if claim == nil { 
            if volume.Status.Phase != v1.VolumeReleased && volume.Status.Phase != v1.VolumeFailed {
                // Also, log this only once:
                klog.V(2).Infof("volume %q is released and reclaim policy %q will be executed", volume.Name, volume.Spec.PersistentVolumeReclaimPolicy)
                if volume, err = ctrl.updateVolumePhase(volume, v1.VolumeReleased, ""); err != nil { 
                    return err
                }
            }
            //根据persistentVolumeReclaimPolicy配置做相应的处理，Retain 保留/ Delete 删除/ Recycle 回收
            if err = ctrl.reclaimVolume(volume); err != nil { 
                return err
            }
            if volume.Spec.PersistentVolumeReclaimPolicy == v1.PersistentVolumeReclaimRetain {
                // volume is being retained, it references a claim that does not exist now.
                klog.V(4).Infof("PersistentVolume[%s] references a claim %q (%s) that is not found", volume.Name, claimrefToClaimKey(volume.Spec.ClaimRef), volume.Spec.ClaimRef.UID)
            }
            return nil
        } else if claim.Spec.VolumeName == "" {
            if pvutil.CheckVolumeModeMismatches(&claim.Spec, &volume.Spec) { 
                volumeMsg := fmt.Sprintf("Cannot bind PersistentVolume to requested PersistentVolumeClaim %q due to incompatible volumeMode.", claim.Name)
                ctrl.eventRecorder.Event(volume, v1.EventTypeWarning, events.VolumeMismatch, volumeMsg)
                claimMsg := fmt.Sprintf("Cannot bind PersistentVolume %q to requested PersistentVolumeClaim due to incompatible volumeMode.", volume.Name)
                ctrl.eventRecorder.Event(claim, v1.EventTypeWarning, events.VolumeMismatch, claimMsg)
                // Skipping syncClaim
                return nil
            }

            if metav1.HasAnnotation(volume.ObjectMeta, pvutil.AnnBoundByController) { 
                klog.V(4).Infof("synchronizing PersistentVolume[%s]: volume not bound yet, waiting for syncClaim to fix it", volume.Name)
            } else { 
                klog.V(4).Infof("synchronizing PersistentVolume[%s]: volume was bound and got unbound (by user?), waiting for syncClaim to fix it", volume.Name)
            } 
            ctrl.claimQueue.Add(claimToClaimKey(claim))
            return nil
        //  已经绑定更新状态status phase为Bound
        } else if claim.Spec.VolumeName == volume.Name {
            // Volume is bound to a claim properly, update status if necessary
            klog.V(4).Infof("synchronizing PersistentVolume[%s]: all is bound", volume.Name)
            if _, err = ctrl.updateVolumePhase(volume, v1.VolumeBound, ""); err != nil {
                // Nothing was saved; we will fall back into the same
                // condition in the next call to this method
                return err
            }
            return nil
        //  PV绑定到PVC上，但是PVC被绑定到其他PV上，重置
        } else {
            // Volume is bound to a claim, but the claim is bound elsewhere
            if metav1.HasAnnotation(volume.ObjectMeta, pvutil.AnnDynamicallyProvisioned) && volume.Spec.PersistentVolumeReclaimPolicy == v1.PersistentVolumeReclaimDelete {

                if volume.Status.Phase != v1.VolumeReleased && volume.Status.Phase != v1.VolumeFailed { 
                    klog.V(2).Infof("dynamically volume %q is released and it will be deleted", volume.Name)
                    if volume, err = ctrl.updateVolumePhase(volume, v1.VolumeReleased, ""); err != nil { 
                        return err
                    }
                }
                if err = ctrl.reclaimVolume(volume); err != nil { 
                    return err
                }
                return nil
            } else { 
                if metav1.HasAnnotation(volume.ObjectMeta, pvutil.AnnBoundByController) { 
                    klog.V(4).Infof("synchronizing PersistentVolume[%s]: volume is bound by controller to a claim that is bound to another volume, unbinding", volume.Name)
                    if err = ctrl.unbindVolume(volume); err != nil {
                        return err
                    }
                    return nil
                } else { 
                    klog.V(4).Infof("synchronizing PersistentVolume[%s]: volume is bound by user to a claim that is bound to another volume, waiting for the claim to get unbound", volume.Name) 
                    if err = ctrl.unbindVolume(volume); err != nil {
                        return err
                    }
                    return nil
                }
            }
        }
    }
}

```

**ctrl.reclaimVolume**

（1）根据pv对象的annotation：pv.kubernetes.io/provisioned-by配置，决定走in-tree或out-tree逻辑来删除存储、删除pv对象。

（2）out-tree逻辑：让相应的plugin来删除存储，并删除pv对象（例：当volume plugin为ceph-csi时，由external provisioner来完成删除存储与pv对象的操作）。

（3）in-tree逻辑：在ctrl.doDeleteVolume方法中进行删除存储的操作，在ctrl.deleteVolumeOperation方法中进行删除pv对象的操作。

```

```

**3 ctrl.claimWorker**

claimWorker主要是维护pvc的状态，根据不同的状况对pvc对象的状态值进行更新，主要是给pvc找到合适的pv做绑定bound操作；当找不到合适的pv时，会调用volume plugin来创建底层存储，并创建pv对象（这里包含了in-tree和out-tree两条逻辑）。



claimWorker会不断循环消费claimQueue队列里面的数据，然后获取到相应的PVC执行updateClaim方法做进一步处理。

**3.1 ctrl.updateClaim**

主要调用ctrl.syncClaim做进一步处理。

**3.1.1 ctrl.syncClaim**

主要逻辑：

（1）检查pvc中key为"pv.kubernetes.io/bind-completed"的annotation，有则说明该pvc已经完成了绑定操作；

（2）没有该annotation，则调用ctrl.syncUnboundClaim，给Unbound的pvc，找到对应的PV，执行绑定操作；

（3）有该annotation，则调用ctrl.syncBoundClaim，看是否需要做修复逻辑。



**ctrl.syncUnboundClaim主要逻辑：**

（1）调用ctrl.volumes.findBestMatchForClaim，给Unbound的pvc，找到对应的PV，调用bind执行绑定操作；

（2）当找不到合适的pv时，调用ctrl.provisionClaim来做进一步操作。



ctrl.volumes.findBestMatchForClaim主要逻辑：给pvc找到合适的pv列表。怎么找呢，其实一开始介绍pvc如何选择合适的pv来绑定的时候就有介绍过，具体逻辑可以自己看一下代码，pv与pvc提前绑定特性也在该方法里有所体现。



**ctrl.provisionClaim主要逻辑：**

（1）获取pvc对应的storageclass对象，根据volume plugin的配置，决定走in-tree或out-tree逻辑来创建存储、创建pv对象。

（2）out-tree逻辑：调用ctrl.provisionClaimOperationExternal来给pvc对象设置annotation：volume.beta.kubernetes.io/storage-provisioner：{plugin name}，让相应的plugin来创建存储，并创建pv对象（例：当volume plugin为ceph-csi时，由external provisioner来完成创建存储与pv对象的操作）。

（3）in-tree逻辑：调用ctrl.provisionClaimOperation来创建存储，并创建pv对象。

```
//kubernetes/pkg/controller/volume/persistentvolume/pv_controller.go

func (ctrl *PersistentVolumeController) syncUnboundClaim(claim *v1.PersistentVolumeClaim) error {
	// This is a new PVC that has not completed binding
	// OBSERVATION: pvc is "Pending"
	if claim.Spec.VolumeName == "" {
		// User did not care which PV they get.
		delayBinding, err := pvutil.IsDelayBindingMode(claim, ctrl.classLister)
		if err != nil {
			return err
		}

		// [Unit test set 1]
		volume, err := ctrl.volumes.findBestMatchForClaim(claim, delayBinding)
		if err != nil {
			klog.V(2).Infof("synchronizing unbound PersistentVolumeClaim[%s]: Error finding PV for claim: %v", claimToClaimKey(claim), err)
			return fmt.Errorf("Error finding PV for claim %q: %v", claimToClaimKey(claim), err)
		}
		if volume == nil {
			klog.V(4).Infof("synchronizing unbound PersistentVolumeClaim[%s]: no volume found", claimToClaimKey(claim))
			// No PV could be found
			// OBSERVATION: pvc is "Pending", will retry
			switch {
			case delayBinding && !pvutil.IsDelayBindingProvisioning(claim):
				ctrl.eventRecorder.Event(claim, v1.EventTypeNormal, events.WaitForFirstConsumer, "waiting for first consumer to be created before binding")
			case v1helper.GetPersistentVolumeClaimClass(claim) != "":
				if err = ctrl.provisionClaim(claim); err != nil {
					return err
				}
				return nil
			default:
				ctrl.eventRecorder.Event(claim, v1.EventTypeNormal, events.FailedBinding, "no persistent volumes available for this claim and no storage class is set")
			}

			// Mark the claim as Pending and try to find a match in the next
			// periodic syncClaim
			if _, err = ctrl.updateClaimStatus(claim, v1.ClaimPending, nil); err != nil {
				return err
			}
			return nil
		} else /* pv != nil */ {
			// Found a PV for this claim
			// OBSERVATION: pvc is "Pending", pv is "Available"
			claimKey := claimToClaimKey(claim)
			klog.V(4).Infof("synchronizing unbound PersistentVolumeClaim[%s]: volume %q found: %s", claimKey, volume.Name, getVolumeStatusForLogging(volume))
			if err = ctrl.bind(volume, claim); err != nil {
				// On any error saving the volume or the claim, subsequent
				// syncClaim will finish the binding.
				// record count error for provision if exists
				// timestamp entry will remain in cache until a success binding has happened
				metrics.RecordMetric(claimKey, &ctrl.operationTimestamps, err)
				return err
			}
			// OBSERVATION: claim is "Bound", pv is "Bound"
			// if exists a timestamp entry in cache, record end to end provision latency and clean up cache
			// End of the provision + binding operation lifecycle, cache will be cleaned by "RecordMetric"
			// [Unit test 12-1, 12-2, 12-4]
			metrics.RecordMetric(claimKey, &ctrl.operationTimestamps, nil)
			return nil
		}
	} else /* pvc.Spec.VolumeName != nil */ {
		// [Unit test set 2]
		// User asked for a specific PV.
		klog.V(4).Infof("synchronizing unbound PersistentVolumeClaim[%s]: volume %q requested", claimToClaimKey(claim), claim.Spec.VolumeName)
		obj, found, err := ctrl.volumes.store.GetByKey(claim.Spec.VolumeName)
		if err != nil {
			return err
		}
		if !found {
			// User asked for a PV that does not exist.
			// OBSERVATION: pvc is "Pending"
			// Retry later.
			klog.V(4).Infof("synchronizing unbound PersistentVolumeClaim[%s]: volume %q requested and not found, will try again next time", claimToClaimKey(claim), claim.Spec.VolumeName)
			if _, err = ctrl.updateClaimStatus(claim, v1.ClaimPending, nil); err != nil {
				return err
			}
			return nil
		} else {
			volume, ok := obj.(*v1.PersistentVolume)
			if !ok {
				return fmt.Errorf("Cannot convert object from volume cache to volume %q!?: %+v", claim.Spec.VolumeName, obj)
			}
			klog.V(4).Infof("synchronizing unbound PersistentVolumeClaim[%s]: volume %q requested and found: %s", claimToClaimKey(claim), claim.Spec.VolumeName, getVolumeStatusForLogging(volume))
			if volume.Spec.ClaimRef == nil {
				// User asked for a PV that is not claimed
				// OBSERVATION: pvc is "Pending", pv is "Available"
				klog.V(4).Infof("synchronizing unbound PersistentVolumeClaim[%s]: volume is unbound, binding", claimToClaimKey(claim))
				if err = checkVolumeSatisfyClaim(volume, claim); err != nil {
					klog.V(4).Infof("Can't bind the claim to volume %q: %v", volume.Name, err)
					// send an event
					msg := fmt.Sprintf("Cannot bind to requested volume %q: %s", volume.Name, err)
					ctrl.eventRecorder.Event(claim, v1.EventTypeWarning, events.VolumeMismatch, msg)
					// volume does not satisfy the requirements of the claim
					if _, err = ctrl.updateClaimStatus(claim, v1.ClaimPending, nil); err != nil {
						return err
					}
				} else if err = ctrl.bind(volume, claim); err != nil {
					// On any error saving the volume or the claim, subsequent
					// syncClaim will finish the binding.
					return err
				}
				// OBSERVATION: pvc is "Bound", pv is "Bound"
				return nil
			} else if pvutil.IsVolumeBoundToClaim(volume, claim) {
				// User asked for a PV that is claimed by this PVC
				// OBSERVATION: pvc is "Pending", pv is "Bound"
				klog.V(4).Infof("synchronizing unbound PersistentVolumeClaim[%s]: volume already bound, finishing the binding", claimToClaimKey(claim))

				// Finish the volume binding by adding claim UID.
				if err = ctrl.bind(volume, claim); err != nil {
					return err
				}
				// OBSERVATION: pvc is "Bound", pv is "Bound"
				return nil
			} else {
				// User asked for a PV that is claimed by someone else
				// OBSERVATION: pvc is "Pending", pv is "Bound"
				if !metav1.HasAnnotation(claim.ObjectMeta, pvutil.AnnBoundByController) {
					klog.V(4).Infof("synchronizing unbound PersistentVolumeClaim[%s]: volume already bound to different claim by user, will retry later", claimToClaimKey(claim))
					// User asked for a specific PV, retry later
					if _, err = ctrl.updateClaimStatus(claim, v1.ClaimPending, nil); err != nil {
						return err
					}
					return nil
				} else {
					// This should never happen because someone had to remove
					// AnnBindCompleted annotation on the claim.
					klog.V(4).Infof("synchronizing unbound PersistentVolumeClaim[%s]: volume already bound to different claim %q by controller, THIS SHOULD NEVER HAPPEN", claimToClaimKey(claim), claimrefToClaimKey(volume.Spec.ClaimRef))
					return fmt.Errorf("Invalid binding of claim %q to volume %q: volume already claimed by %q", claimToClaimKey(claim), claim.Spec.VolumeName, claimrefToClaimKey(volume.Spec.ClaimRef))
				}
			}
		}
	}
}

// provisionClaim starts new asynchronous operation to provision a claim if
// provisioning is enabled.
func (ctrl *PersistentVolumeController) provisionClaim(claim *v1.PersistentVolumeClaim) error {
	if !ctrl.enableDynamicProvisioning {
		return nil
	}
	klog.V(4).Infof("provisionClaim[%s]: started", claimToClaimKey(claim))
	opName := fmt.Sprintf("provision-%s[%s]", claimToClaimKey(claim), string(claim.UID))
	plugin, storageClass, err := ctrl.findProvisionablePlugin(claim)
	// findProvisionablePlugin does not return err for external provisioners
	if err != nil {
		ctrl.eventRecorder.Event(claim, v1.EventTypeWarning, events.ProvisioningFailed, err.Error())
		klog.Errorf("error finding provisioning plugin for claim %s: %v", claimToClaimKey(claim), err)
		// failed to find the requested provisioning plugin, directly return err for now.
		// controller will retry the provisioning in every syncUnboundClaim() call
		// retain the original behavior of returning nil from provisionClaim call
		return nil
	}
	ctrl.scheduleOperation(opName, func() error {
		// create a start timestamp entry in cache for provision operation if no one exists with
		// key = claimKey, pluginName = provisionerName, operation = "provision"
		claimKey := claimToClaimKey(claim)
		ctrl.operationTimestamps.AddIfNotExist(claimKey, ctrl.getProvisionerName(plugin, storageClass), "provision")
		var err error
		if plugin == nil {
			_, err = ctrl.provisionClaimOperationExternal(claim, storageClass)
		} else {
			_, err = ctrl.provisionClaimOperation(claim, plugin, storageClass)
		}
		// if error happened, record an error count metric
		// timestamp entry will remain in cache until a success binding has happened
		if err != nil {
			metrics.RecordMetric(claimKey, &ctrl.operationTimestamps, err)
		}
		return err
	})
	return nil
}

```

至此，pv controller的分析已经完毕，下面进行一下简单的总结。



**总结**

PV Cotroller全称PersistentVolume controller，主要负责pv、pvc对象的绑定与pv、pvc对象的生命周期管理（如创建/删除底层存储，创建/删除pv对象，pv与pvc对象的状态变更）。



创建存储、pv对象

当一个pvc对象创建出来后，pv controller会为其寻找合适的pv进行绑定，当一个 PVC 找不到合适的 PV 时，相应的volume plugin就会根据 StorageClass对象的参数配置去做一个动态创建 PV 的操作。



根据pvc对应的storageclass对象中volume plugin的配置，决定走in-tree或out-tree逻辑来创建存储、创建pv对象：

（1）out-tree逻辑：pv controller来给pvc对象设置annotation：volume.beta.kubernetes.io/storage-provisioner：{plugin name}，让相应的plugin来创建存储，并创建pv对象（例：当volume plugin为ceph-csi时，由external provisioner来完成创建存储与pv对象的操作）。

（2）in-tree逻辑：调用内置的volume plungin来创建存储，并创建pv对象。



删除存储、pv对象

当与pv绑定的pvc被删除后，pv的状态变更为released，如果pv配置的回收策略为retain，则不会对pv以及底层存储资源做删除操作，如果pv的回收策略为delete，则调用volume plugin来做底层存储以及pv对象的删除操作。（动态创建的pv，其配置的回收策略继承自storageclass对象）



根据pv对象的annotation：pv.kubernetes.io/provisioned-by配置，决定走in-tree或out-tree逻辑来删除存储、删除pv对象：

（1）out-tree逻辑：让相应的plugin来删除存储，并删除pv对象（例：当volume plugin为ceph-csi时，由external provisioner来完成删除存储与pv对象的操作）。

（2）in-tree逻辑：pv controller调用内置的volume plungin来删除存储，并创建pv对象。



