**概述**

接下来将对external-provisioner组件进行源码分析。

在external-provisioner组件中，rbd与cephfs共用一套处理逻辑，也即同一套代码，同时适用于rbd存储与cephfs存储。

external-provisioner组件的源码分析分为三部分：

（1）主体处理逻辑分析；

（2）main方法与Leader选举分析；

（3）组件启动参数分析。

基于tag v1.6.0

[https://github.com/kubernetes-csi/external-provisioner/releases/tag/v1.6.0](https://github.com/kubernetes-csi/external-provisioner/releases/tag/v1.6.0)

**external-provisioner作用介绍**

（1）create pvc时，external-provisioner参与存储资源与pv对象的创建。external-provisioner组件监听到pvc创建事件后，负责拼接请求，调用ceph-csi组件的CreateVolume方法来创建存储，创建存储成功后，创建pv对象；

（2）delete pvc时，external-provisioner参与存储资源与pv对象的删除。当pvc被删除时，pv controller会将其绑定的pv对象状态由bound更新为release，external-provisioner监听到pv更新事件后，调用ceph-csi的DeleteVolume方法来删除存储，并删除pv对象。

external-provisioner源码分析（1）-主体处理逻辑分析

external-provisioner组件中，主要的业务处理逻辑都在provisionController中了，所以对external-provisioner组件的分析，先从provisionController入手。

provisionController主要负责处理claimQueue（也即处理pvc对象的新增与更新事件），根据需要调用ceph-csi组件的CreateVolume方法来创建存储，并创建pv对象；与处理volumeQueue（也即处理pv对象的新增与更新事件），根据pv的状态以及回收策略决定是否调用ceph-csi组件的DeleteVolume方法来删除存储，并删除pv对象。

后续会对claimQueue与volumeQueue进行分析。

main方法中调用了provisionController.Run\(wait.NeverStop\)，作为provisionController的分析入口。

**provisionController.Run\(\)**

provisionController.Run\(\)中定义了run方法并执行。主要关注run方法中的ctrl.runClaimWorker与ctrl.runVolumeWorker，这两个方法负责处理主体逻辑。

```
// Run starts all of this controller's control loops
func (ctrl *ProvisionController) Run(_ <-chan struct{}) {
    // TODO: arg is as of 1.12 unused. Nothing can ever be cancelled. Should
    // accept a context instead and use it instead of context.TODO(), but would
    // break API. Not urgent: realistically, users are simply passing in
    // wait.NeverStop() anyway.

    run := func(ctx context.Context) {
        glog.Infof("Starting provisioner controller %s!", ctrl.component)
        defer utilruntime.HandleCrash()
        defer ctrl.claimQueue.ShutDown()
        defer ctrl.volumeQueue.ShutDown()

        ctrl.hasRunLock.Lock()
        ctrl.hasRun = true
        ctrl.hasRunLock.Unlock()
        if ctrl.metricsPort > 0 {
            prometheus.MustRegister([]prometheus.Collector{
                metrics.PersistentVolumeClaimProvisionTotal,
                metrics.PersistentVolumeClaimProvisionFailedTotal,
                metrics.PersistentVolumeClaimProvisionDurationSeconds,
                metrics.PersistentVolumeDeleteTotal,
                metrics.PersistentVolumeDeleteFailedTotal,
                metrics.PersistentVolumeDeleteDurationSeconds,
            }...)
            http.Handle(ctrl.metricsPath, promhttp.Handler())
            address := net.JoinHostPort(ctrl.metricsAddress, strconv.FormatInt(int64(ctrl.metricsPort), 10))
            glog.Infof("Starting metrics server at %s\n", address)
            go wait.Forever(func() {
                err := http.ListenAndServe(address, nil)
                if err != nil {
                    glog.Errorf("Failed to listen on %s: %v", address, err)
                }
            }, 5*time.Second)
        }

        // If a external SharedInformer has been passed in, this controller
        // should not call Run again
        if !ctrl.customClaimInformer {
            go ctrl.claimInformer.Run(ctx.Done())
        }
        if !ctrl.customVolumeInformer {
            go ctrl.volumeInformer.Run(ctx.Done())
        }
        if !ctrl.customClassInformer {
            go ctrl.classInformer.Run(ctx.Done())
        }

        if !cache.WaitForCacheSync(ctx.Done(), ctrl.claimInformer.HasSynced, ctrl.volumeInformer.HasSynced, ctrl.classInformer.HasSynced) {
            return
        }

        // 两个worker跑多个goroutine
        for i := 0; i < ctrl.threadiness; i++ {
            go wait.Until(ctrl.runClaimWorker, time.Second, context.TODO().Done())
            go wait.Until(ctrl.runVolumeWorker, time.Second, context.TODO().Done())
        }

        glog.Infof("Started provisioner controller %s!", ctrl.component)

        select {}
    }

    go ctrl.volumeStore.Run(context.TODO(), DefaultThreadiness)

    // 选主相关
    if ctrl.leaderElection {
        rl, err := resourcelock.New("endpoints",
            ctrl.leaderElectionNamespace,
            strings.Replace(ctrl.provisionerName, "/", "-", -1),
            ctrl.client.CoreV1(),
            nil,
            resourcelock.ResourceLockConfig{
                Identity:      ctrl.id,
                EventRecorder: ctrl.eventRecorder,
            })
        if err != nil {
            glog.Fatalf("Error creating lock: %v", err)
        }

        leaderelection.RunOrDie(context.TODO(), leaderelection.LeaderElectionConfig{
            Lock:          rl,
            LeaseDuration: ctrl.leaseDuration,
            RenewDeadline: ctrl.renewDeadline,
            RetryPeriod:   ctrl.retryPeriod,
            Callbacks: leaderelection.LeaderCallbacks{
                OnStartedLeading: run,
                OnStoppedLeading: func() {
                    glog.Fatalf("leaderelection lost")
                },
            },
        })
        panic("unreachable")
    } else {
        run(context.TODO())
    }
}
```

接下来将分别对run方法中的ctrl.runClaimWorker与ctrl.runVolumeWorker进行分析。

**1.ctrl.runClaimWorker**

根据threadiness的个数，起相应个数的goroutine，运行ctrl.runClaimWorker。

主要负责处理claimQueue，处理pvc对象的新增与更新事件，根据需要调用csi组件的CreateVolume方法来创建存储，并创建pv对象。

```
        for i := 0; i < ctrl.threadiness; i++ {
            go wait.Until(ctrl.runClaimWorker, time.Second, context.TODO().Done())
            go wait.Until(ctrl.runVolumeWorker, time.Second, context.TODO().Done())
        }
        
// vendor/sigs.k8s.io/sig-storage-lib-external-provisioner/v5/controller/controller.go

func (ctrl *ProvisionController) runClaimWorker() {
    // 无限循环processNextClaimWorkItem
	for ctrl.processNextClaimWorkItem() {
	}
}

```

调用链：main\(\) --&gt; provisionController.Run\(\) --&gt; ctrl.runClaimWorker\(\) --&gt; ctrl.processNextClaimWorkItem\(\) --&gt; ctrl.syncClaimHandler\(\) --&gt; ctrl.syncClaim\(\) --&gt; ctrl.provisionClaimOperation\(\) --&gt; ctrl.provisioner.Provision\(\)



**1.1 ctrl.processNextClaimWorkItem**

主要逻辑：

（1）从claimQueue中获取pvc；

（2）调ctrl.syncClaimHandler做进一步处理；

（3）处理成功后，清理该pvc的rateLimiter，并将pvc从claimsInProgress中移除；

（4）处理失败后，会进行一定次数的重试，即将该pvc添加rateLimiter；

（5）最后，无论调ctrl.syncClaimHandler成功与否，将该pvc从claimQueue中移除。

```
// Map UID -> *PVC with all claims that may be provisioned in the background.
claimsInProgress sync.Map

// processNextClaimWorkItem processes items from claimQueue
func (ctrl *ProvisionController) processNextClaimWorkItem() bool {
    // 从claimQueue中获取pvc
	obj, shutdown := ctrl.claimQueue.Get()

	if shutdown {
		return false
	}

	err := func(obj interface{}) error {
	    // 最后，无论调ctrl.syncClaimHandler成功与否，将该pvc从claimQueue中移除
		defer ctrl.claimQueue.Done(obj)
		var key string
		var ok bool
		if key, ok = obj.(string); !ok {
			ctrl.claimQueue.Forget(obj)
			return fmt.Errorf("expected string in workqueue but got %#v", obj)
		}
        
        // 调ctrl.syncClaimHandler做进一步处理
		if err := ctrl.syncClaimHandler(key); err != nil {
		    // 处理失败后，会进行一定次数的重试，即将该pvc添加rateLimiter
			if ctrl.failedProvisionThreshold == 0 {
				glog.Warningf("Retrying syncing claim %q, failure %v", key, ctrl.claimQueue.NumRequeues(obj))
				ctrl.claimQueue.AddRateLimited(obj)
			} else if ctrl.claimQueue.NumRequeues(obj) < ctrl.failedProvisionThreshold {
				glog.Warningf("Retrying syncing claim %q because failures %v < threshold %v", key, ctrl.claimQueue.NumRequeues(obj), ctrl.failedProvisionThreshold)
				ctrl.claimQueue.AddRateLimited(obj)
			} else {
				glog.Errorf("Giving up syncing claim %q because failures %v >= threshold %v", key, ctrl.claimQueue.NumRequeues(obj), ctrl.failedProvisionThreshold)
				glog.V(2).Infof("Removing PVC %s from claims in progress", key)
				ctrl.claimsInProgress.Delete(key) // This can leak a volume that's being provisioned in the background!
				// Done but do not Forget: it will not be in the queue but NumRequeues
				// will be saved until the obj is deleted from kubernetes
			}
			return fmt.Errorf("error syncing claim %q: %s", key, err.Error())
		}
        
        // 处理成功后，清理该pvc的rateLimiter，并将pvc从claimsInProgress中移除
		ctrl.claimQueue.Forget(obj)
		// Silently remove the PVC from list of volumes in progress. The provisioning either succeeded
		// or the PVC was ignored by this provisioner.
		ctrl.claimsInProgress.Delete(key)
		return nil
	}(obj)

	if err != nil {
		utilruntime.HandleError(err)
		return true
	}

	return true
}

```

下面先分析一下claimQueue的相关方法：

  


（1）Done：从claimQueue中删除

```
// Done marks item as done processing, and if it has been marked as dirty again
// while it was being processed, it will be re-added to the queue for
// re-processing.
func (q *Type) Done(item interface{}) {
	q.cond.L.Lock()
	defer q.cond.L.Unlock()

	q.metrics.done(item)

	q.processing.delete(item)
	if q.dirty.has(item) {
		q.queue = append(q.queue, item)
		q.cond.Signal()
	}
}

```

（2）Forget：仅清理rateLimiter

```
func (q *rateLimitingType) Forget(item interface{}) {
	q.rateLimiter.Forget(item)
}

```

（3）AddRateLimited：在速率限制器表示可以之后向claimQueue重新加入

```
// AddRateLimited AddAfter's the item based on the time when the rate limiter says it's ok
func (q *rateLimitingType) AddRateLimited(item interface{}) {
	q.DelayingInterface.AddAfter(item, q.rateLimiter.When(item))
}

// AddAfter adds the given item to the work queue after the given delay
func (q *delayingType) AddAfter(item interface{}, duration time.Duration) {
	// don't add if we're already shutting down
	if q.ShuttingDown() {
		return
	}

	q.metrics.retry()

	// immediately add things with no delay
	if duration <= 0 {
		q.Add(item)
		return
	}

	select {
	case <-q.stopCh:
		// unblock if ShutDown() is called
	case q.waitingForAddCh <- &waitFor{data: item, readyAt: q.clock.Now().Add(duration)}:
	}
}

```

#### 1.1.1 ctrl.syncClaimHandler

最主要是调ctrl.syncClaim

```
// syncClaimHandler gets the claim from informer's cache then calls syncClaim. A non-nil error triggers requeuing of the claim.
func (ctrl *ProvisionController) syncClaimHandler(key string) error {
	objs, err := ctrl.claimsIndexer.ByIndex(uidIndex, key)
	if err != nil {
		return err
	}
	var claimObj interface{}
	if len(objs) > 0 {
		claimObj = objs[0]
	} else {
		obj, found := ctrl.claimsInProgress.Load(key)
		if !found {
			utilruntime.HandleError(fmt.Errorf("claim %q in work queue no longer exists", key))
			return nil
		}
		claimObj = obj
	}
	return ctrl.syncClaim(claimObj)
}

```

###### 1.1.1.1 ctrl.syncClaim

主要逻辑：  
（1）先调用ctrl.shouldProvision判断是否需要provision操作；  
（2）调ctrl.provisionClaimOperation做进一步操作。

```
// syncClaim checks if the claim should have a volume provisioned for it and
// provisions one if so. Returns an error if the claim is to be requeued.
func (ctrl *ProvisionController) syncClaim(obj interface{}) error {
	claim, ok := obj.(*v1.PersistentVolumeClaim)
	if !ok {
		return fmt.Errorf("expected claim but got %+v", obj)
	}

	should, err := ctrl.shouldProvision(claim)
	if err != nil {
		ctrl.updateProvisionStats(claim, err, time.Time{})
		return err
	} else if should {
		startTime := time.Now()

		status, err := ctrl.provisionClaimOperation(claim)
		ctrl.updateProvisionStats(claim, err, startTime)
		if err == nil || status == ProvisioningFinished {
			// Provisioning is 100% finished / not in progress.
			switch err {
			case nil:
				glog.V(5).Infof("Claim processing succeeded, removing PVC %s from claims in progress", claim.UID)
			case errStopProvision:
				glog.V(5).Infof("Stop provisioning, removing PVC %s from claims in progress", claim.UID)
				// Our caller would requeue if we pass on this special error; return nil instead.
				err = nil
			default:
				glog.V(2).Infof("Final error received, removing PVC %s from claims in progress", claim.UID)
			}
			ctrl.claimsInProgress.Delete(string(claim.UID))
			return err
		}
		if status == ProvisioningInBackground {
			// Provisioning is in progress in background.
			glog.V(2).Infof("Temporary error received, adding PVC %s to claims in progress", claim.UID)
			ctrl.claimsInProgress.Store(string(claim.UID), claim)
		} else {
			// status == ProvisioningNoChange.
			// Don't change claimsInProgress:
			// - the claim is already there if previous status was ProvisioningInBackground.
			// - the claim is not there if if previous status was ProvisioningFinished.
		}
		return err
	}
	return nil
}

```

**ctrl.shouldProvision**

该方法主要判断一个pvc对象是否需要进行provision操作，主要逻辑：

（1）当claim.Spec.VolumeName不为空时，不需要进行provision操作，返回false；

（2）调qualifier.ShouldProvision判断存储driver是否支持provision操作；

（3）如果是Kubernetes 1.5及以上版本，则从pvc的annotation：pv.kubernetes.io/provisioned-by中获取driver名称，并判断该driver名称是否与该external-provisioner组件的driver名称一致，一致则说明该pvc将由该external-provisioner组件进行底层存储的创建以及pv对象的创建，不一致则不做处理，直接返回。

```
// shouldProvision returns whether a claim should have a volume provisioned for
// it, i.e. whether a Provision is "desired"
func (ctrl *ProvisionController) shouldProvision(claim *v1.PersistentVolumeClaim) (bool, error) {
	if claim.Spec.VolumeName != "" {
		return false, nil
	}

	if qualifier, ok := ctrl.provisioner.(Qualifier); ok {
		if !qualifier.ShouldProvision(claim) {
			return false, nil
		}
	}

	// Kubernetes 1.5 provisioning with annStorageProvisioner
	if ctrl.kubeVersion.AtLeast(utilversion.MustParseSemantic("v1.5.0")) {
		if provisioner, found := claim.Annotations[annStorageProvisioner]; found {
			if ctrl.knownProvisioner(provisioner) {
				claimClass := util.GetPersistentVolumeClaimClass(claim)
				class, err := ctrl.getStorageClass(claimClass)
				if err != nil {
					return false, err
				}
				if class.VolumeBindingMode != nil && *class.VolumeBindingMode == storage.VolumeBindingWaitForFirstConsumer {
					// When claim is in delay binding mode, annSelectedNode is
					// required to provision volume.
					// Though PV controller set annStorageProvisioner only when
					// annSelectedNode is set, but provisioner may remove
					// annSelectedNode to notify scheduler to reschedule again.
					if selectedNode, ok := claim.Annotations[annSelectedNode]; ok && selectedNode != "" {
						return true, nil
					}
					return false, nil
				}
				return true, nil
			}
		}
	} else {
		// Kubernetes 1.4 provisioning, evaluating class.Provisioner
		claimClass := util.GetPersistentVolumeClaimClass(claim)
		class, err := ctrl.getStorageClass(claimClass)
		if err != nil {
			glog.Errorf("Error getting claim %q's StorageClass's fields: %v", claimToClaimKey(claim), err)
			return false, err
		}
		if class.Provisioner != ctrl.provisionerName {
			return false, nil
		}

		return true, nil
	}

	return false, nil
}

```

**ctrl.provisionClaimOperation**

主要逻辑：

（1）从pvc对象中获取storageclass的名称；

（2）拼接pv名称（格式：pvc-{pvc对象的uid}）；

（3）检查pv是否已经存在，存在则直接返回；

（4）从pvc对象中获取信息构造claimRef（用于后续拼接pv对象）；

（5）检查是否支持动态创建存储；

（6）获取storageclass对象；

（7）检查storageclass对象中的provisioner是否已注册；

（8）构造options结构体；

（9）开始调用provision方法，返回pv对象结构体；

（10）pv对象结构体额外信息添加；

（11）发送创建pv对象的请求给apiserver。

    // provisionClaimOperation attempts to provision a volume for the given claim.
    // Returns nil error only when the volume was provisioned (in which case it also returns ProvisioningFinished),
    // a normal error when the volume was not provisioned and provisioning should be retried (requeue the claim),
    // or the special errStopProvision when provisioning was impossible and no further attempts to provision should be tried.
    func (ctrl *ProvisionController) provisionClaimOperation(claim *v1.PersistentVolumeClaim) (ProvisioningState, error) {
    	// Most code here is identical to that found in controller.go of kube's PV controller...

    	// 从pvc对象中获取storageclass的名称
    	claimClass := util.GetPersistentVolumeClaimClass(claim)
    	operation := fmt.Sprintf("provision %q class %q", claimToClaimKey(claim), claimClass)
    	glog.Info(logOperation(operation, "started"))

    	//  检查pv是否已经存在
    	pvName := ctrl.getProvisionedVolumeNameForClaim(claim)
    	volume, err := ctrl.client.CoreV1().PersistentVolumes().Get(pvName, metav1.GetOptions{})
    	if err == nil && volume != nil {
    		// Volume has been already provisioned, nothing to do.
    		glog.Info(logOperation(operation, "persistentvolume %q already exists, skipping", pvName))
    		return ProvisioningFinished, errStopProvision
    	}

    	// 从pvc对象中获取信息构造claimRef
    	claimRef, err := ref.GetReference(scheme.Scheme, claim)
    	if err != nil {
    		glog.Error(logOperation(operation, "unexpected error getting claim reference: %v", err))
    		return ProvisioningNoChange, err
    	}

    	// 检查是否支持动态创建存储
    	if err = ctrl.canProvision(claim); err != nil {
    		ctrl.eventRecorder.Event(claim, v1.EventTypeWarning, "ProvisioningFailed", err.Error())
    		glog.Error(logOperation(operation, "failed to provision volume: %v", err))
    		return ProvisioningFinished, errStopProvision
    	}

    	// 获取storageclass对象
    	class, err := ctrl.getStorageClass(claimClass)
    	if err != nil {
    		glog.Error(logOperation(operation, "error getting claim's StorageClass's fields: %v", err))
    		return ProvisioningFinished, err
    	}

    	// 检查storageclass对象中的provisioner是否已注册
    	if !ctrl.knownProvisioner(class.Provisioner) {
    		// class.Provisioner has either changed since shouldProvision() or
    		// annDynamicallyProvisioned contains different provisioner than
    		// class.Provisioner.
    		glog.Error(logOperation(operation, "unknown provisioner %q requested in claim's StorageClass", class.Provisioner))
    		return ProvisioningFinished, errStopProvision
    	}

        // 指定节点相关操作
    	var selectedNode *v1.Node
    	if ctrl.kubeVersion.AtLeast(utilversion.MustParseSemantic("v1.11.0")) {
    		// Get SelectedNode
    		if nodeName, ok := getString(claim.Annotations, annSelectedNode, annAlphaSelectedNode); ok {
    			selectedNode, err = ctrl.client.CoreV1().Nodes().Get(nodeName, metav1.GetOptions{}) // TODO (verult) cache Nodes
    			if err != nil {
    				err = fmt.Errorf("failed to get target node: %v", err)
    				ctrl.eventRecorder.Event(claim, v1.EventTypeWarning, "ProvisioningFailed", err.Error())
    				return ProvisioningNoChange, err
    			}
    		}
    	}

        // 构造options结构体
    	options := ProvisionOptions{
    		StorageClass: class,
    		PVName:       pvName,
    		PVC:          claim,
    		SelectedNode: selectedNode,
    	}

    	ctrl.eventRecorder.Event(claim, v1.EventTypeNormal, "Provisioning", fmt.Sprintf("External provisioner is provisioning volume for claim %q", claimToClaimKey(claim)))

        // 开始调用provision方法，返回pv对象结构体
    	result := ProvisioningFinished
    	if p, ok := ctrl.provisioner.(ProvisionerExt); ok {
    		volume, result, err = p.ProvisionExt(options)
    	} else {
    		volume, err = ctrl.provisioner.Provision(options)
    	}
    	if err != nil {
    		if ierr, ok := err.(*IgnoredError); ok {
    			// Provision ignored, do nothing and hope another provisioner will provision it.
    			glog.Info(logOperation(operation, "volume provision ignored: %v", ierr))
    			return ProvisioningFinished, errStopProvision
    		}
    		err = fmt.Errorf("failed to provision volume with StorageClass %q: %v", claimClass, err)
    		ctrl.eventRecorder.Event(claim, v1.EventTypeWarning, "ProvisioningFailed", err.Error())
    		if _, ok := claim.Annotations[annSelectedNode]; ok && result == ProvisioningReschedule {
    			// For dynamic PV provisioning with delayed binding, the provisioner may fail
    			// because the node is wrong (permanent error) or currently unusable (not enough
    			// capacity). If the provisioner wants to give up scheduling with the currently
    			// selected node, then it can ask for that by returning ProvisioningReschedule
    			// as state.
    			//
    			// `selectedNode` must be removed to notify scheduler to schedule again.
    			if errLabel := ctrl.rescheduleProvisioning(claim); errLabel != nil {
    				glog.Info(logOperation(operation, "volume rescheduling failed: %v", errLabel))
    				// If unsetting that label fails in ctrl.rescheduleProvisioning, we
    				// keep the volume in the work queue as if the provisioner had
    				// returned ProvisioningFinished and simply try again later.
    				return ProvisioningFinished, err
    			}
    			// Label was removed, stop working on the volume.
    			glog.Info(logOperation(operation, "volume rescheduled because: %v", err))
    			return ProvisioningFinished, errStopProvision
    		}

    		// ProvisioningReschedule shouldn't have been returned for volumes without selected node,
    		// but if we get it anyway, then treat it like ProvisioningFinished because we cannot
    		// reschedule.
    		if result == ProvisioningReschedule {
    			result = ProvisioningFinished
    		}
    		return result, err
    	}

    	glog.Info(logOperation(operation, "volume %q provisioned", volume.Name))

        // pv对象结构体额外信息添加
    	// Set ClaimRef and the PV controller will bind and set annBoundByController for us
    	volume.Spec.ClaimRef = claimRef

    	// Add external provisioner finalizer if it doesn't already have it
    	if ctrl.addFinalizer && !ctrl.checkFinalizer(volume, finalizerPV) {
    		volume.ObjectMeta.Finalizers = append(volume.ObjectMeta.Finalizers, finalizerPV)
    	}

    	metav1.SetMetaDataAnnotation(&volume.ObjectMeta, annDynamicallyProvisioned, ctrl.provisionerName)
    	if ctrl.kubeVersion.AtLeast(utilversion.MustParseSemantic("v1.6.0")) {
    		volume.Spec.StorageClassName = claimClass
    	} else {
    		metav1.SetMetaDataAnnotation(&volume.ObjectMeta, annClass, claimClass)
    	}

    	glog.Info(logOperation(operation, "succeeded"))

        // 创建pv到apiserver
    	if err := ctrl.volumeStore.StoreVolume(claim, volume); err != nil {
    		return ProvisioningFinished, err
    	}
    	return ProvisioningFinished, nil
    }


**ctrl.provisioner.Provision：**



Provision方法主要是调用csi的CreateVolume方法来创建存储，并返回pv对象结构体。



主要逻辑：

（1）调用p.checkDriverCapabilities检测driver提供的能力；

（2）构建pv对象名称；

（3）从StorageClass.Parameters中获取fstype并校验；

（4）从pvc中获取存储申请大小；

（5）构建CreateVolumeRequest结构体；

（6）获取provisioner、controllerPublish、nodeStage、nodePublish、controllerExpand操作对应的secret的名称与命名空间；

（7）从apiserver获取provisioner对应的secret对象，存放进CreateVolumeRequest结构体；

（8）去除部分StorageClass.Parameters中的参数，然后存放进CreateVolumeRequest结构体；

（9）调用p.csiClient.CreateVolume（也即调用ceph-csi的CreateVolume方法）来创建ceph存储；

（10）存储创建成功后，校验创建的大小是否符合需求；

（11）构建pv对象结构体并返回。

```
func (p *csiProvisioner) Provision(options controller.ProvisionOptions) (*v1.PersistentVolume, error) {
	// The controller should call ProvisionExt() instead, but just in case...
	pv, _, err := p.ProvisionExt(options)
	return pv, err
}

func (p *csiProvisioner) ProvisionExt(options controller.ProvisionOptions) (*v1.PersistentVolume, controller.ProvisioningState, error) {
	if options.StorageClass == nil {
		return nil, controller.ProvisioningFinished, errors.New("storage class was nil")
	}

	if options.PVC.Annotations[annStorageProvisioner] != p.driverName && options.PVC.Annotations[annMigratedTo] != p.driverName {
		// The storage provisioner annotation may not equal driver name but the
		// PVC could have annotation "migrated-to" which is the new way to
		// signal a PVC is migrated (k8s v1.17+)
		return nil, controller.ProvisioningFinished, &controller.IgnoredError{
			Reason: fmt.Sprintf("PVC annotated with external-provisioner name %s does not match provisioner driver name %s. This could mean the PVC is not migrated",
				options.PVC.Annotations[annStorageProvisioner],
				p.driverName),
		}

	}

	migratedVolume := false
	if p.supportsMigrationFromInTreePluginName != "" {
		// NOTE: we cannot depend on PVC.Annotations[volume.beta.kubernetes.io/storage-provisioner] to get
		// the in-tree provisioner name in case of CSI migration scenarios. The annotation will be
		// set to the CSI provisioner name by PV controller for migration scenarios
		// so that external provisioner can correctly pick up the PVC pointing to an in-tree plugin
		if options.StorageClass.Provisioner == p.supportsMigrationFromInTreePluginName {
			klog.V(2).Infof("translating storage class for in-tree plugin %s to CSI", options.StorageClass.Provisioner)
			storageClass, err := p.translator.TranslateInTreeStorageClassToCSI(p.supportsMigrationFromInTreePluginName, options.StorageClass)
			if err != nil {
				return nil, controller.ProvisioningFinished, fmt.Errorf("failed to translate storage class: %v", err)
			}
			options.StorageClass = storageClass
			migratedVolume = true
		} else {
			klog.V(4).Infof("skip translation of storage class for plugin: %s", options.StorageClass.Provisioner)
		}
	}

	// Make sure the plugin is capable of fulfilling the requested options
	rc := &requiredCapabilities{}
	if options.PVC.Spec.DataSource != nil {
		// PVC.Spec.DataSource.Name is the name of the VolumeSnapshot API object
		if options.PVC.Spec.DataSource.Name == "" {
			return nil, controller.ProvisioningFinished, fmt.Errorf("the PVC source not found for PVC %s", options.PVC.Name)
		}

		switch options.PVC.Spec.DataSource.Kind {
		case snapshotKind:
			if *(options.PVC.Spec.DataSource.APIGroup) != snapshotAPIGroup {
				return nil, controller.ProvisioningFinished, fmt.Errorf("the PVC source does not belong to the right APIGroup. Expected %s, Got %s", snapshotAPIGroup, *(options.PVC.Spec.DataSource.APIGroup))
			}
			rc.snapshot = true
		case pvcKind:
			rc.clone = true
		default:
			klog.Infof("Unsupported DataSource specified (%s), the provisioner won't act on this request", options.PVC.Spec.DataSource.Kind)
		}
	}
	if err := p.checkDriverCapabilities(rc); err != nil {
		return nil, controller.ProvisioningFinished, err
	}

	if options.PVC.Spec.Selector != nil {
		return nil, controller.ProvisioningFinished, fmt.Errorf("claim Selector is not supported")
	}

	pvName, err := makeVolumeName(p.volumeNamePrefix, fmt.Sprintf("%s", options.PVC.ObjectMeta.UID), p.volumeNameUUIDLength)
	if err != nil {
		return nil, controller.ProvisioningFinished, err
	}

	fsTypesFound := 0
	fsType := ""
	for k, v := range options.StorageClass.Parameters {
		if strings.ToLower(k) == "fstype" || k == prefixedFsTypeKey {
			fsType = v
			fsTypesFound++
		}
		if strings.ToLower(k) == "fstype" {
			klog.Warningf(deprecationWarning("fstype", prefixedFsTypeKey, ""))
		}
	}
	if fsTypesFound > 1 {
		return nil, controller.ProvisioningFinished, fmt.Errorf("fstype specified in parameters with both \"fstype\" and \"%s\" keys", prefixedFsTypeKey)
	}
	if len(fsType) == 0 {
		fsType = defaultFSType
	}

	capacity := options.PVC.Spec.Resources.Requests[v1.ResourceName(v1.ResourceStorage)]
	volSizeBytes := capacity.Value()

	// add by zhongjialiang, if it is rbd request and fstype is xfs, we will check volume request size, it need to be equal or larger than 1G
	// because if the request size is less than 1G, it may occur an error when kubelet call NodeStageVolume(ceph-csi)
	// failed to run mkfs error: exit status 1, output: agsize (xxxx blocks) too small, need at least 4096 blocks
	//
	// issue: https://git-sa.nie.netease.com/venice/ceph-csi/issues/105
	klog.Infof("request volume size is %s (volSizeBytes: %d)", capacity.String(), volSizeBytes)
	if isRbdRequest(options.StorageClass.Parameters) && fsType == xfsFSType && volSizeBytes < oneGi {
		return nil, controller.ProvisioningFinished, fmt.Errorf("fstype xfs volume size request must be equal or larger than 1Gi, but your volume size request is %s ", capacity.String())
	}

	// Get access mode
	volumeCaps := make([]*csi.VolumeCapability, 0)
	for _, pvcAccessMode := range options.PVC.Spec.AccessModes {
		volumeCaps = append(volumeCaps, getVolumeCapability(options, pvcAccessMode, fsType))
	}

	// Create a CSI CreateVolumeRequest and Response
	req := csi.CreateVolumeRequest{
		Name:               pvName,
		Parameters:         options.StorageClass.Parameters,
		VolumeCapabilities: volumeCaps,
		CapacityRange: &csi.CapacityRange{
			RequiredBytes: int64(volSizeBytes),
		},
	}

	if options.PVC.Spec.DataSource != nil && (rc.clone || rc.snapshot) {
		volumeContentSource, err := p.getVolumeContentSource(options)
		if err != nil {
			return nil, controller.ProvisioningNoChange, fmt.Errorf("error getting handle for DataSource Type %s by Name %s: %v", options.PVC.Spec.DataSource.Kind, options.PVC.Spec.DataSource.Name, err)
		}
		req.VolumeContentSource = volumeContentSource
	}

	if options.PVC.Spec.DataSource != nil && rc.clone {
		err = p.setCloneFinalizer(options.PVC)
		if err != nil {
			return nil, controller.ProvisioningNoChange, err
		}
	}

	if p.supportsTopology() {
		requirements, err := GenerateAccessibilityRequirements(
			p.client,
			p.driverName,
			options.PVC.Name,
			options.StorageClass.AllowedTopologies,
			options.SelectedNode,
			p.strictTopology,
			p.csiNodeLister,
			p.nodeLister)
		if err != nil {
			return nil, controller.ProvisioningNoChange, fmt.Errorf("error generating accessibility requirements: %v", err)
		}
		req.AccessibilityRequirements = requirements
	}

	klog.V(5).Infof("CreateVolumeRequest %+v", req)

	rep := &csi.CreateVolumeResponse{}

	// Resolve provision secret credentials.
	provisionerSecretRef, err := getSecretReference(provisionerSecretParams, options.StorageClass.Parameters, pvName, options.PVC)
	if err != nil {
		return nil, controller.ProvisioningNoChange, err
	}
	provisionerCredentials, err := getCredentials(p.client, provisionerSecretRef)
	if err != nil {
		return nil, controller.ProvisioningNoChange, err
	}
	req.Secrets = provisionerCredentials

	// Resolve controller publish, node stage, node publish secret references
	controllerPublishSecretRef, err := getSecretReference(controllerPublishSecretParams, options.StorageClass.Parameters, pvName, options.PVC)
	if err != nil {
		return nil, controller.ProvisioningNoChange, err
	}
	nodeStageSecretRef, err := getSecretReference(nodeStageSecretParams, options.StorageClass.Parameters, pvName, options.PVC)
	if err != nil {
		return nil, controller.ProvisioningNoChange, err
	}
	nodePublishSecretRef, err := getSecretReference(nodePublishSecretParams, options.StorageClass.Parameters, pvName, options.PVC)
	if err != nil {
		return nil, controller.ProvisioningNoChange, err
	}
	controllerExpandSecretRef, err := getSecretReference(controllerExpandSecretParams, options.StorageClass.Parameters, pvName, options.PVC)
	if err != nil {
		return nil, controller.ProvisioningNoChange, err
	}

	req.Parameters, err = removePrefixedParameters(options.StorageClass.Parameters)
	if err != nil {
		return nil, controller.ProvisioningFinished, fmt.Errorf("failed to strip CSI Parameters of prefixed keys: %v", err)
	}

	if p.extraCreateMetadata {
		// add pvc and pv metadata to request for use by the plugin
		req.Parameters[pvcNameKey] = options.PVC.GetName()
		req.Parameters[pvcNamespaceKey] = options.PVC.GetNamespace()
		req.Parameters[pvNameKey] = pvName
	}

	ctx, cancel := context.WithTimeout(context.Background(), p.timeout)
	defer cancel()
	rep, err = p.csiClient.CreateVolume(ctx, &req)

	if err != nil {
		// Giving up after an error and telling the pod scheduler to retry with a different node
		// only makes sense if:
		// - The CSI driver supports topology: without that, the next CreateVolume call after
		//   rescheduling will be exactly the same.
		// - We are working on a volume with late binding: only in that case will
		//   provisioning be retried if we give up for now.
		// - The error is one where rescheduling is
		//   a) allowed (i.e. we don't have to keep calling CreateVolume because the operation might be running) and
		//   b) it makes sense (typically local resource exhausted).
		//   isFinalError is going to check this.
		//
		// We do this regardless whether the driver has asked for strict topology because
		// even drivers which did not ask for it explicitly might still only look at the first
		// topology entry and thus succeed after rescheduling.
		mayReschedule := p.supportsTopology() &&
			options.SelectedNode != nil
		state := checkError(err, mayReschedule)
		klog.V(5).Infof("CreateVolume failed, supports topology = %v, node selected %v => may reschedule = %v => state = %v: %v",
			p.supportsTopology(),
			options.SelectedNode != nil,
			mayReschedule,
			state,
			err)
		return nil, state, err
	}

	if rep.Volume != nil {
		klog.V(3).Infof("create volume rep: %+v", *rep.Volume)
	}
	volumeAttributes := map[string]string{provisionerIDKey: p.identity}
	for k, v := range rep.Volume.VolumeContext {
		volumeAttributes[k] = v
	}
	respCap := rep.GetVolume().GetCapacityBytes()

	//According to CSI spec CreateVolume should be able to return capacity = 0, which means it is unknown. for example NFS/FTP
	if respCap == 0 {
		respCap = volSizeBytes
		klog.V(3).Infof("csiClient response volume with size 0, which is not supported by apiServer, will use claim size:%d", respCap)
	} else if respCap < volSizeBytes {
		capErr := fmt.Errorf("created volume capacity %v less than requested capacity %v", respCap, volSizeBytes)
		delReq := &csi.DeleteVolumeRequest{
			VolumeId: rep.GetVolume().GetVolumeId(),
		}
		err = cleanupVolume(p, delReq, provisionerCredentials)
		if err != nil {
			capErr = fmt.Errorf("%v. Cleanup of volume %s failed, volume is orphaned: %v", capErr, pvName, err)
		}
		// use InBackground to retry the call, hoping the volume is deleted correctly next time.
		return nil, controller.ProvisioningInBackground, capErr
	}

	if options.PVC.Spec.DataSource != nil {
		contentSource := rep.GetVolume().ContentSource
		if contentSource == nil {
			sourceErr := fmt.Errorf("volume content source missing")
			delReq := &csi.DeleteVolumeRequest{
				VolumeId: rep.GetVolume().GetVolumeId(),
			}
			err = cleanupVolume(p, delReq, provisionerCredentials)
			if err != nil {
				sourceErr = fmt.Errorf("%v. cleanup of volume %s failed, volume is orphaned: %v", sourceErr, pvName, err)
			}
			return nil, controller.ProvisioningInBackground, sourceErr
		}
	}

	pv := &v1.PersistentVolume{
		ObjectMeta: metav1.ObjectMeta{
			Name: pvName,
		},
		Spec: v1.PersistentVolumeSpec{
			AccessModes:  options.PVC.Spec.AccessModes,
			MountOptions: options.StorageClass.MountOptions,
			Capacity: v1.ResourceList{
				v1.ResourceName(v1.ResourceStorage): bytesToGiQuantity(respCap),
			},
			// TODO wait for CSI VolumeSource API
			PersistentVolumeSource: v1.PersistentVolumeSource{
				CSI: &v1.CSIPersistentVolumeSource{
					Driver:                     p.driverName,
					VolumeHandle:               p.volumeIdToHandle(rep.Volume.VolumeId),
					VolumeAttributes:           volumeAttributes,
					ControllerPublishSecretRef: controllerPublishSecretRef,
					NodeStageSecretRef:         nodeStageSecretRef,
					NodePublishSecretRef:       nodePublishSecretRef,
					ControllerExpandSecretRef:  controllerExpandSecretRef,
				},
			},
		},
	}

	if options.StorageClass.ReclaimPolicy != nil {
		pv.Spec.PersistentVolumeReclaimPolicy = *options.StorageClass.ReclaimPolicy
	}

	if p.supportsTopology() {
		pv.Spec.NodeAffinity = GenerateVolumeNodeAffinity(rep.Volume.AccessibleTopology)
	}

	// Set VolumeMode to PV if it is passed via PVC spec when Block feature is enabled
	if options.PVC.Spec.VolumeMode != nil {
		pv.Spec.VolumeMode = options.PVC.Spec.VolumeMode
	}
	// Set FSType if PV is not Block Volume
	if !util.CheckPersistentVolumeClaimModeBlock(options.PVC) {
		pv.Spec.PersistentVolumeSource.CSI.FSType = fsType
	}

	klog.V(2).Infof("successfully created PV %v for PVC %v and csi volume name %v", pv.Name, options.PVC.Name, pv.Spec.CSI.VolumeHandle)

	if migratedVolume {
		pv, err = p.translator.TranslateCSIPVToInTree(pv)
		if err != nil {
			klog.Warningf("failed to translate CSI PV to in-tree due to: %v. Deleting provisioned PV", err)
			deleteErr := p.Delete(pv)
			if deleteErr != nil {
				klog.Warningf("failed to delete partly provisioned PV: %v", deleteErr)
				// Retry the call again to clean up the orphan
				return nil, controller.ProvisioningInBackground, err
			}
			return nil, controller.ProvisioningFinished, err
		}
	}

	klog.V(5).Infof("successfully created PV %+v", pv.Spec.PersistentVolumeSource)
	return pv, controller.ProvisioningFinished, nil
}

```

**2.ctrl.runVolumeWorker**

根据threadiness的个数，起相应个数的goroutine，运行ctrl.runVolumeWorker。



主要负责处理volumeQueue，处理pv对象的新增与更新事件，根据需要调用csi组件的DeleteVolume方法来删除存储，并删除pv对象）。

```
        for i := 0; i < ctrl.threadiness; i++ {
			go wait.Until(ctrl.runClaimWorker, time.Second, context.TODO().Done())
			go wait.Until(ctrl.runVolumeWorker, time.Second, context.TODO().Done())
		}
// vendor/sigs.k8s.io/sig-storage-lib-external-provisioner/v5/controller/controller.go

func (ctrl *ProvisionController) runClaimWorker() {
    // 无限循环ctrl.processNextClaimWorkItem
	for ctrl.processNextClaimWorkItem() {
	}
}

```

调用链：main\(\) --&gt; provisionController.Run\(\) --&gt; ctrl.runVolumeWorker\(\) --&gt; ctrl.processNextVolumeWorkItem\(\) --&gt; ctrl.syncVolumeHandler\(\) --&gt; ctrl.syncVolume\(\) --&gt; ctrl.deleteVolumeOperation\(\) --&gt; ctrl.provisioner.Delete\(\)



**2.1 crtl.processNextVolumeWorkItem**

主要逻辑：

（1）从volumeQueue中获取pv；

（2）调ctrl.syncClaimHandler做进一步处理；

（3）处理成功后，清理该pv的rateLimiter；

（4）处理失败后，会进行一定次数的重试，即将该pv添加rateLimiter；

（5）最后，无论调ctrl.syncVolumeHandler成功与否，将该pv从volumeQueue中移除。



主要是调ctrl.syncVolumeHandler

```
// processNextVolumeWorkItem processes items from volumeQueue
func (ctrl *ProvisionController) processNextVolumeWorkItem() bool {
    // 从volumeQueue中获取pv
	obj, shutdown := ctrl.volumeQueue.Get()

	if shutdown {
		return false
	}

	err := func(obj interface{}) error {
	    // 最后，无论调ctrl.syncVolumeHandler成功与否，将该pv从volumeQueue中移除
		defer ctrl.volumeQueue.Done(obj)
		var key string
		var ok bool
		if key, ok = obj.(string); !ok {
			ctrl.volumeQueue.Forget(obj)
			return fmt.Errorf("expected string in workqueue but got %#v", obj)
		}
        
        // 调ctrl.syncVolumeHandler做进一步处理
		if err := ctrl.syncVolumeHandler(key); err != nil {
		    // 处理失败后，会进行一定次数的重试，即将该pv添加rateLimiter
			if ctrl.failedDeleteThreshold == 0 {
				glog.Warningf("Retrying syncing volume %q, failure %v", key, ctrl.volumeQueue.NumRequeues(obj))
				ctrl.volumeQueue.AddRateLimited(obj)
			} else if ctrl.volumeQueue.NumRequeues(obj) < ctrl.failedDeleteThreshold {
				glog.Warningf("Retrying syncing volume %q because failures %v < threshold %v", key, ctrl.volumeQueue.NumRequeues(obj), ctrl.failedDeleteThreshold)
				ctrl.volumeQueue.AddRateLimited(obj)
			} else {
				glog.Errorf("Giving up syncing volume %q because failures %v >= threshold %v", key, ctrl.volumeQueue.NumRequeues(obj), ctrl.failedDeleteThreshold)
				// Done but do not Forget: it will not be in the queue but NumRequeues
				// will be saved until the obj is deleted from kubernetes
			}
			return fmt.Errorf("error syncing volume %q: %s", key, err.Error())
		}
        
        // 处理成功后，清理该pv的rateLimiter
		ctrl.volumeQueue.Forget(obj)
		return nil
	}(obj)

	if err != nil {
		utilruntime.HandleError(err)
		return true
	}

	return true
}

```

#### 2.1.1 ctrl.syncVolumeHandler

主要是调ctrl.syncVolume

```
// syncVolumeHandler gets the volume from informer's cache then calls syncVolume
func (ctrl *ProvisionController) syncVolumeHandler(key string) error {
	volumeObj, exists, err := ctrl.volumes.GetByKey(key)
	if err != nil {
		return err
	}
	if !exists {
		utilruntime.HandleError(fmt.Errorf("volume %q in work queue no longer exists", key))
		return nil
	}

	return ctrl.syncVolume(volumeObj)
}

```

###### 2.1.1.1 ctrl.syncVolume

主要逻辑：  
（1）调ctrl.shouldDelete判断是否要进行删除动作；  
（2）要进行删除动作，则调ctrl.deleteVolumeOperation。

```
// syncVolume checks if the volume should be deleted and deletes if so
func (ctrl *ProvisionController) syncVolume(obj interface{}) error {
	volume, ok := obj.(*v1.PersistentVolume)
	if !ok {
		return fmt.Errorf("expected volume but got %+v", obj)
	}

	if ctrl.shouldDelete(volume) {
		startTime := time.Now()
		err := ctrl.deleteVolumeOperation(volume)
		ctrl.updateDeleteStats(volume, err, startTime)
		return err
	}
	return nil
}

```

**ctrl.shouldDelete**

主要逻辑：

（1）判断provisioner是否授权删除动作；

（2）1.9+版本k8s中，做PV protection相关校验（Finalizer相关判断）；

（3）1.5+版本k8s中，当pv的state不是Released或Failed时，返回false；

（4）判断pv的回收策略是否为delete，不是则返回false；

（5）从pvc的annotation：pv.kubernetes.io/provisioned-by中获取driver名称，并判断该driver名称是否与该external-provisioner组件的driver名称一致，一致则说明该pvc将由该external-provisioner组件进行底层存储的删除以及pv对象的删除，不一致则不做处理，直接返回。

```
// shouldDelete returns whether a volume should have its backing volume
// deleted, i.e. whether a Delete is "desired"
func (ctrl *ProvisionController) shouldDelete(volume *v1.PersistentVolume) bool {
    // 判断provisioner是否授权删除动作
	if deletionGuard, ok := ctrl.provisioner.(DeletionGuard); ok {
		if !deletionGuard.ShouldDelete(volume) {
			return false
		}
	}

	// In 1.9+ PV protection means the object will exist briefly with a
	// deletion timestamp even after our successful Delete. Ignore it.
	if ctrl.kubeVersion.AtLeast(utilversion.MustParseSemantic("v1.9.0")) {
		if ctrl.addFinalizer && !ctrl.checkFinalizer(volume, finalizerPV) && volume.ObjectMeta.DeletionTimestamp != nil {
			return false
		} else if volume.ObjectMeta.DeletionTimestamp != nil {
			return false
		}
	}

	// In 1.5+ we delete only if the volume is in state Released. In 1.4 we must
	// delete if the volume is in state Failed too.
	if ctrl.kubeVersion.AtLeast(utilversion.MustParseSemantic("v1.5.0")) {
		if volume.Status.Phase != v1.VolumeReleased {
			return false
		}
	} else {
		if volume.Status.Phase != v1.VolumeReleased && volume.Status.Phase != v1.VolumeFailed {
			return false
		}
	}
    
    // 判断回收策略是否为delete，不是则返回false
	if volume.Spec.PersistentVolumeReclaimPolicy != v1.PersistentVolumeReclaimDelete {
		return false
	}

	if !metav1.HasAnnotation(volume.ObjectMeta, annDynamicallyProvisioned) {
		return false
	}

	ann := volume.Annotations[annDynamicallyProvisioned]
	migratedTo := volume.Annotations[annMigratedTo]
	if ann != ctrl.provisionerName && migratedTo != ctrl.provisionerName {
		return false
	}

	return true
}

```

###### 2.1.1.2 crtl.deleteVolumeOperation

主要逻辑：  
（1）调provisioner.Delete删除rbd image；  
（2）从apiserver中删除pv对象；  
（3）Finalizer相关处理。

```
// deleteVolumeOperation attempts to delete the volume backing the given
// volume. Returns error, which indicates whether deletion should be retried
// (requeue the volume) or not
func (ctrl *ProvisionController) deleteVolumeOperation(volume *v1.PersistentVolume) error {
	operation := fmt.Sprintf("delete %q", volume.Name)
	glog.Info(logOperation(operation, "started"))

	// This method may have been waiting for a volume lock for some time.
	// Our check does not have to be as sophisticated as PV controller's, we can
	// trust that the PV controller has set the PV to Released/Failed and it's
	// ours to delete
	newVolume, err := ctrl.client.CoreV1().PersistentVolumes().Get(volume.Name, metav1.GetOptions{})
	if err != nil {
		return nil
	}
	if !ctrl.shouldDelete(newVolume) {
		glog.Info(logOperation(operation, "persistentvolume no longer needs deletion, skipping"))
		return nil
	}

	err = ctrl.provisioner.Delete(volume)
	if err != nil {
		if ierr, ok := err.(*IgnoredError); ok {
			// Delete ignored, do nothing and hope another provisioner will delete it.
			glog.Info(logOperation(operation, "volume deletion ignored: %v", ierr))
			return nil
		}
		// Delete failed, emit an event.
		glog.Error(logOperation(operation, "volume deletion failed: %v", err))
		ctrl.eventRecorder.Event(volume, v1.EventTypeWarning, "VolumeFailedDelete", err.Error())
		return err
	}

	glog.Info(logOperation(operation, "volume deleted"))

	// Delete the volume
	if err = ctrl.client.CoreV1().PersistentVolumes().Delete(volume.Name, nil); err != nil {
		// Oops, could not delete the volume and therefore the controller will
		// try to delete the volume again on next update.
		glog.Info(logOperation(operation, "failed to delete persistentvolume: %v", err))
		return err
	}

	if ctrl.addFinalizer {
		if len(newVolume.ObjectMeta.Finalizers) > 0 {
			// Remove external-provisioner finalizer

			// need to get the pv again because the delete has updated the object with a deletion timestamp
			newVolume, err := ctrl.client.CoreV1().PersistentVolumes().Get(volume.Name, metav1.GetOptions{})
			if err != nil {
				// If the volume is not found return, otherwise error
				if !apierrs.IsNotFound(err) {
					glog.Info(logOperation(operation, "failed to get persistentvolume to update finalizer: %v", err))
					return err
				}
				return nil
			}
			finalizers := make([]string, 0)
			for _, finalizer := range newVolume.ObjectMeta.Finalizers {
				if finalizer != finalizerPV {
					finalizers = append(finalizers, finalizer)
				}
			}

			// Only update the finalizers if we actually removed something
			if len(finalizers) != len(newVolume.ObjectMeta.Finalizers) {
				newVolume.ObjectMeta.Finalizers = finalizers
				if _, err = ctrl.client.CoreV1().PersistentVolumes().Update(newVolume); err != nil {
					if !apierrs.IsNotFound(err) {
						// Couldn't remove finalizer and the object still exists, the controller may
						// try to remove the finalizer again on the next update
						glog.Info(logOperation(operation, "failed to remove finalizer for persistentvolume: %v", err))
						return err
					}
				}
			}
		}
	}

	glog.Info(logOperation(operation, "persistentvolume deleted"))

	glog.Info(logOperation(operation, "succeeded"))
	return nil
}

```

ctrl.provisioner.Delete主要是调ceph-csi组件的DeleteVolume来删除存储。



主要逻辑：

（1）获取volumeId；

（2）构建DeleteVolumeRequest请求结构体；

（3）从apiserver中获取删除存储所需的secret对象，并存放进DeleteVolumeRequest请求结构体；

（4）调用p.csiClient.DeleteVolume（也即调用ceph-csi的DeleteVolume方法）来删除存储。

```
func (p *csiProvisioner) Delete(volume *v1.PersistentVolume) error {
	if volume == nil {
		return fmt.Errorf("invalid CSI PV")
	}

	var err error
	if p.translator.IsPVMigratable(volume) {
		// we end up here only if CSI migration is enabled in-tree (both overall
		// and for the specific plugin that is migratable) causing in-tree PV
		// controller to yield deletion of PVs with in-tree source to external provisioner
		// based on AnnDynamicallyProvisioned annotation.
		volume, err = p.translator.TranslateInTreePVToCSI(volume)
		if err != nil {
			return err
		}
	}

	if volume.Spec.CSI == nil {
		return fmt.Errorf("invalid CSI PV")
	}

	volumeId := p.volumeHandleToId(volume.Spec.CSI.VolumeHandle)

	rc := &requiredCapabilities{}
	if err := p.checkDriverCapabilities(rc); err != nil {
		return err
	}

	req := csi.DeleteVolumeRequest{
		VolumeId: volumeId,
	}

	// get secrets if StorageClass specifies it
	storageClassName := util.GetPersistentVolumeClass(volume)
	if len(storageClassName) != 0 {
		if storageClass, err := p.scLister.Get(storageClassName); err == nil {

			// edit by zhongjialiang
			// issue: https://git-sa.nie.netease.com/venice/ceph-csi/issues/85
			secretParams := provisionerSecretParams

			if isRbdRequest(storageClass.Parameters) {
				secretParams = deleteSecretParams
			}

			// Resolve provision secret credentials.
			provisionerSecretRef, err := getSecretReference(secretParams, storageClass.Parameters, volume.Name, &v1.PersistentVolumeClaim{
				ObjectMeta: metav1.ObjectMeta{
					Name:      volume.Spec.ClaimRef.Name,
					Namespace: volume.Spec.ClaimRef.Namespace,
				},
			})
			if err != nil {
				return fmt.Errorf("failed to get secretreference for volume %s: %v", volume.Name, err)
			}

			credentials, err := getCredentials(p.client, provisionerSecretRef)
			if err != nil {
				// Continue with deletion, as the secret may have already been deleted.
				klog.Errorf("Failed to get credentials for volume %s: %s", volume.Name, err.Error())
			}
			req.Secrets = credentials
		} else {
			klog.Warningf("failed to get storageclass: %s, proceeding to delete without secrets. %v", storageClassName, err)
		}
	}
	ctx, cancel := context.WithTimeout(context.Background(), p.timeout)
	defer cancel()

	_, err = p.csiClient.DeleteVolume(ctx, &req)

	return err
}

```

到这里，对external-provisioner的provisionController的分析就结束了，下面将对provisionController的分析做一个简单的总结。



总结

provisionController主要负责处理以下两个队列：



（1）claimQueue

存放pvc对象的新增与更新事件，provisionController处理该队列时只处理annotation：pv.kubernetes.io/provisioned-by中的值与external-provisioner组件的driver名称一致的pvc对象，对符合条件的pvc对象，调用ceph-csi组件的CreateVolume方法来创建存储，并创建pv对象；



（2）volumeQueue

存放pv对象的新增与更新事件，provisionController处理该队列时只处理annotation：pv.kubernetes.io/provisioned-by中的值与external-provisioner组件的driver名称一致的pvc对象，根据pv的状态以及回收策略决定是否调用ceph-csi组件的DeleteVolume方法来删除存储，并删除pv对象。



