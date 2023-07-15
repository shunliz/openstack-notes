概述

接下来将对external-provisioner组件进行源码分析。



在external-provisioner组件中，rbd与cephfs共用一套处理逻辑，也即同一套代码，同时适用于rbd存储与cephfs存储。



external-provisioner组件的源码分析分为三部分：

（1）主体处理逻辑分析；

（2）main方法与Leader选举分析；

（3）组件启动参数分析。



基于tag v1.6.0

https://github.com/kubernetes-csi/external-provisioner/releases/tag/v1.6.0



external-provisioner作用介绍

（1）create pvc时，external-provisioner参与存储资源与pv对象的创建。external-provisioner组件监听到pvc创建事件后，负责拼接请求，调用ceph-csi组件的CreateVolume方法来创建存储，创建存储成功后，创建pv对象；

（2）delete pvc时，external-provisioner参与存储资源与pv对象的删除。当pvc被删除时，pv controller会将其绑定的pv对象状态由bound更新为release，external-provisioner监听到pv更新事件后，调用ceph-csi的DeleteVolume方法来删除存储，并删除pv对象。



external-provisioner源码分析（1）-主体处理逻辑分析

external-provisioner组件中，主要的业务处理逻辑都在provisionController中了，所以对external-provisioner组件的分析，先从provisionController入手。



provisionController主要负责处理claimQueue（也即处理pvc对象的新增与更新事件），根据需要调用ceph-csi组件的CreateVolume方法来创建存储，并创建pv对象；与处理volumeQueue（也即处理pv对象的新增与更新事件），根据pv的状态以及回收策略决定是否调用ceph-csi组件的DeleteVolume方法来删除存储，并删除pv对象。



后续会对claimQueue与volumeQueue进行分析。



main方法中调用了provisionController.Run\(wait.NeverStop\)，作为provisionController的分析入口。



provisionController.Run\(\)

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



1.ctrl.runClaimWorker

根据threadiness的个数，起相应个数的goroutine，运行ctrl.runClaimWorker。



主要负责处理claimQueue，处理pvc对象的新增与更新事件，根据需要调用csi组件的CreateVolume方法来创建存储，并创建pv对象。

```
        for i := 0; i < ctrl.threadiness; i++ {
			go wait.Until(ctrl.runClaimWorker, time.Second, context.TODO().Done())
			go wait.Until(ctrl.runVolumeWorker, time.Second, context.TODO().Done())
		}

```



