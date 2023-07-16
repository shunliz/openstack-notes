## external-provisioner源码分析（2）-main方法与Leader选举分析

关联链接

external-provisioner组件的源码分析分为三部分：

（1）主体处理逻辑分析；

（2）main方法与Leader选举分析；

（3）组件启动参数分析。

1.main方法分析

主要对main方法的主要逻辑进行分析，以及分析下组件的EventHandler，看该组件list/watch哪些对象，对象事件来了怎么处理，以及claimQueue与volumeQueue的对象来源。

main方法主要逻辑分析

main方法主要逻辑：

（1）解析启动参数；

（2）根据配置建立clientset；

（3）与csi driver建立连接，建立grpc client；

（4）进行grpc探测（探测csi driver服务是否准备好），直至探测成功；

（5）与csi driver进行通信，获取driver名称与能力；

（6）根据clientset建立informers；

（7）构建provisionController对象；

（8）定义run方法（包括了provisionController.Run）；

（9）根据--enable-leader-election组件启动参数配置决定是否开启Leader 选举，当不开启时，直接运行run方法，开启时调用le.Run\(\)。

```
func main() {
	var config *rest.Config
	var err error

	flag.Var(utilflag.NewMapStringBool(&featureGates), "feature-gates", "A set of key=value pairs that describe feature gates for alpha/experimental features. "+
		"Options are:\n"+strings.Join(utilfeature.DefaultFeatureGate.KnownFeatures(), "\n"))

	klog.InitFlags(nil)
	flag.CommandLine.AddGoFlagSet(goflag.CommandLine)
	flag.Set("logtostderr", "true")
	flag.Parse()

	if err := utilfeature.DefaultMutableFeatureGate.SetFromMap(featureGates); err != nil {
		klog.Fatal(err)
	}

	if *showVersion {
		fmt.Println(os.Args[0], version)
		os.Exit(0)
	}
	klog.Infof("Version: %s", version)

	// get the KUBECONFIG from env if specified (useful for local/debug cluster)
	kubeconfigEnv := os.Getenv("KUBECONFIG")

	if kubeconfigEnv != "" {
		klog.Infof("Found KUBECONFIG environment variable set, using that..")
		kubeconfig = &kubeconfigEnv
	}

	if *master != "" || *kubeconfig != "" {
		klog.Infof("Either master or kubeconfig specified. building kube config from that..")
		config, err = clientcmd.BuildConfigFromFlags(*master, *kubeconfig)
	} else {
		klog.Infof("Building kube configs for running in cluster...")
		config, err = rest.InClusterConfig()
	}
	if err != nil {
		klog.Fatalf("Failed to create config: %v", err)
	}
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		klog.Fatalf("Failed to create client: %v", err)
	}
	// snapclientset.NewForConfig creates a new Clientset for VolumesnapshotV1beta1Client
	snapClient, err := snapclientset.NewForConfig(config)
	if err != nil {
		klog.Fatalf("Failed to create snapshot client: %v", err)
	}

	// The controller needs to know what the server version is because out-of-tree
	// provisioners aren't officially supported until 1.5
	serverVersion, err := clientset.Discovery().ServerVersion()
	if err != nil {
		klog.Fatalf("Error getting server version: %v", err)
	}

	metricsManager := metrics.NewCSIMetricsManager("" /* driverName */)

	grpcClient, err := ctrl.Connect(*csiEndpoint, metricsManager)
	if err != nil {
		klog.Error(err.Error())
		os.Exit(1)
	}   
	
	// 循环探测，直至CSI driver即cephcsi-rbd服务准备好
	err = ctrl.Probe(grpcClient, *operationTimeout)
	if err != nil {
		klog.Error(err.Error())
		os.Exit(1)
	}

	// 从ceph-csi组件中获取driver name
	provisionerName, err := ctrl.GetDriverName(grpcClient, *operationTimeout)
	if err != nil {
		klog.Fatalf("Error getting CSI driver name: %s", err)
	}
	klog.V(2).Infof("Detected CSI driver %s", provisionerName)
	metricsManager.SetDriverName(provisionerName)
	metricsManager.StartMetricsEndpoint(*metricsAddress, *metricsPath)
    
    // 从ceph-csi组件中获取driver能力
	pluginCapabilities, controllerCapabilities, err := ctrl.GetDriverCapabilities(grpcClient, *operationTimeout)
	if err != nil {
		klog.Fatalf("Error getting CSI driver capabilities: %s", err)
	}

	// Generate a unique ID for this provisioner
	timeStamp := time.Now().UnixNano() / int64(time.Millisecond)
	identity := strconv.FormatInt(timeStamp, 10) + "-" + strconv.Itoa(rand.Intn(10000)) + "-" + provisionerName
    
    // 开始构建infomer
	factory := informers.NewSharedInformerFactory(clientset, ctrl.ResyncPeriodOfCsiNodeInformer)

	// -------------------------------
	// Listers
	// Create informer to prevent hit the API server for all resource request
	scLister := factory.Storage().V1().StorageClasses().Lister()
	claimLister := factory.Core().V1().PersistentVolumeClaims().Lister()

	var csiNodeLister storagelistersv1beta1.CSINodeLister
	var nodeLister v1.NodeLister
	if ctrl.SupportsTopology(pluginCapabilities) {
		csiNodeLister = factory.Storage().V1beta1().CSINodes().Lister()
		nodeLister = factory.Core().V1().Nodes().Lister()
	}

	// -------------------------------
	// PersistentVolumeClaims informer
	rateLimiter := workqueue.NewItemExponentialFailureRateLimiter(*retryIntervalStart, *retryIntervalMax)
	claimQueue := workqueue.NewNamedRateLimitingQueue(rateLimiter, "claims")
	claimInformer := factory.Core().V1().PersistentVolumeClaims().Informer()

	// Setup options
	provisionerOptions := []func(*controller.ProvisionController) error{
		controller.LeaderElection(false), // Always disable leader election in provisioner lib. Leader election should be done here in the CSI provisioner level instead.
		controller.FailedProvisionThreshold(0),
		controller.FailedDeleteThreshold(0),
		controller.RateLimiter(rateLimiter),
		controller.Threadiness(int(*workerThreads)),
		controller.CreateProvisionedPVLimiter(workqueue.DefaultControllerRateLimiter()),
		controller.ClaimsInformer(claimInformer),
	}

	translator := csitrans.New()

	supportsMigrationFromInTreePluginName := ""
	if translator.IsMigratedCSIDriverByName(provisionerName) {
		supportsMigrationFromInTreePluginName, err = translator.GetInTreeNameFromCSIName(provisionerName)
		if err != nil {
			klog.Fatalf("Failed to get InTree plugin name for migrated CSI plugin %s: %v", provisionerName, err)
		}
		klog.V(2).Infof("Supports migration from in-tree plugin: %s", supportsMigrationFromInTreePluginName)
		provisionerOptions = append(provisionerOptions, controller.AdditionalProvisionerNames([]string{supportsMigrationFromInTreePluginName}))
	}

	// Create the provisioner: it implements the Provisioner interface expected by
	// the controller
	csiProvisioner := ctrl.NewCSIProvisioner(
		clientset,
		*operationTimeout,
		identity,
		*volumeNamePrefix,
		*volumeNameUUIDLength,
		grpcClient,
		snapClient,
		provisionerName,
		pluginCapabilities,
		controllerCapabilities,
		supportsMigrationFromInTreePluginName,
		*strictTopology,
		translator,
		scLister,
		csiNodeLister,
		nodeLister,
		claimLister,
		*extraCreateMetadata,
	)

	provisionController = controller.NewProvisionController(
		clientset,
		provisionerName,
		csiProvisioner,
		serverVersion.GitVersion,
		provisionerOptions...,
	)

	csiClaimController := ctrl.NewCloningProtectionController(
		clientset,
		claimLister,
		claimInformer,
		claimQueue,
	)
    
    // 主循环函数
	run := func(context.Context) {
		stopCh := context.Background().Done()
		factory.Start(stopCh)
		cacheSyncResult := factory.WaitForCacheSync(stopCh)
		for _, v := range cacheSyncResult {
			if !v {
				klog.Fatalf("Failed to sync Informers!")
			}
		}
        
        // 跑两个controller，后面主要分析provisionController
		go csiClaimController.Run(int(*finalizerThreads), stopCh)
		provisionController.Run(wait.NeverStop)
	}
    
    // Leader 选举相关
	if !*enableLeaderElection {
		run(context.TODO())
	} else {
		// this lock name pattern is also copied from sigs.k8s.io/sig-storage-lib-external-provisioner/v5/controller
		// to preserve backwards compatibility
		lockName := strings.Replace(provisionerName, "/", "-", -1)
        
        // 使用endpoints或leases资源对象来选leader
		var le leaderElection
		if *leaderElectionType == "endpoints" {
			klog.Warning("The 'endpoints' leader election type is deprecated and will be removed in a future release. Use '--leader-election-type=leases' instead.")
			le = leaderelection.NewLeaderElectionWithEndpoints(clientset, lockName, run)
		} else if *leaderElectionType == "leases" {
			le = leaderelection.NewLeaderElection(clientset, lockName, run)
		} else {
			klog.Error("--leader-election-type must be either 'endpoints' or 'leases'")
			os.Exit(1)
		}

		if *leaderElectionNamespace != "" {
			le.WithNamespace(*leaderElectionNamespace)
		}
        
        // 处理Leader 选举逻辑的方法
		if err := le.Run(); err != nil {
			klog.Fatalf("failed to initialize leader election: %v", err)
		}
	}

}

```

**controller.NewProvisionController**

主要看到EventHandler，定义了该组件list/watch的对象，对象事件来了怎么处理，以及claimQueue与volumeQueue的对象来源。



**claimHandler**

可以看到，claimQueue的来源是pvc对象的新增、更新事件（对claimQueue的处理已在external-provisioner源码分析（1）-主体处理逻辑分析中讲过，忘了的话可以回头看下）。

```
    ...
	// PersistentVolumeClaims

	claimHandler := cache.ResourceEventHandlerFuncs{
		AddFunc:    func(obj interface{}) { controller.enqueueClaim(obj) },
		UpdateFunc: func(oldObj, newObj interface{}) { controller.enqueueClaim(newObj) },
		DeleteFunc: func(obj interface{}) {
			// NOOP. The claim is either in claimsInProgress and in the queue, so it will be processed as usual
			// or it's not in claimsInProgress and then we don't care
		},
	}
	
	if controller.claimInformer != nil {
		controller.claimInformer.AddEventHandlerWithResyncPeriod(claimHandler, controller.resyncPeriod)
	} else {
		controller.claimInformer = informer.Core().V1().PersistentVolumeClaims().Informer()
		controller.claimInformer.AddEventHandler(claimHandler)
	}
	...

// enqueueClaim takes an obj and converts it into UID that is then put onto claim work queue.
func (ctrl *ProvisionController) enqueueClaim(obj interface{}) {
	uid, err := getObjectUID(obj)
	if err != nil {
		utilruntime.HandleError(err)
		return
	}
	if ctrl.claimQueue.NumRequeues(uid) == 0 {
		ctrl.claimQueue.Add(uid)
	}
}

```

#### volumeHandler

可以看到，volumeQueue的来源是pv对象的新增、更新事件（对volumeQueue的处理已在external-provisioner源码分析（1）-主体处理逻辑分析中讲过，忘了的话可以回头看下）。

```
    ...
	// PersistentVolumes
	volumeHandler := cache.ResourceEventHandlerFuncs{
		AddFunc:    func(obj interface{}) { controller.enqueueVolume(obj) },
		UpdateFunc: func(oldObj, newObj interface{}) { controller.enqueueVolume(newObj) },
		DeleteFunc: func(obj interface{}) { controller.forgetVolume(obj) },
	}
	if controller.volumeInformer != nil {
		controller.volumeInformer.AddEventHandlerWithResyncPeriod(volumeHandler, controller.resyncPeriod)
	} else {
		controller.volumeInformer = informer.Core().V1().PersistentVolumes().Informer()
		controller.volumeInformer.AddEventHandler(volumeHandler)
	}
	...

// enqueueVolume takes an obj and converts it into a namespace/name string which
// is then put onto the given work queue.
func (ctrl *ProvisionController) enqueueVolume(obj interface{}) {
	var key string
	var err error
	if key, err = cache.DeletionHandlingMetaNamespaceKeyFunc(obj); err != nil {
		utilruntime.HandleError(err)
		return
	}
	// Re-Adding is harmless but try to add it to the queue only if it is not
	// already there, because if it is already there we *must* be retrying it
	if ctrl.volumeQueue.NumRequeues(key) == 0 {
		ctrl.volumeQueue.Add(key)
	}
}

// forgetVolume Forgets an obj from the given work queue, telling the queue to
// stop tracking its retries because e.g. the obj was deleted
func (ctrl *ProvisionController) forgetVolume(obj interface{}) {
	var key string
	var err error
	if key, err = cache.DeletionHandlingMetaNamespaceKeyFunc(obj); err != nil {
		utilruntime.HandleError(err)
		return
	}
	ctrl.volumeQueue.Forget(key)
	ctrl.volumeQueue.Done(key)
}

```

2.Leader 选举分析

在 Golang 中，k8s client-go 这个package 针对 Leader 相关功能进行了封装，支持3种锁资源，endpoint，configmap，lease，方便使用。



Leader 选举基本原理

Leader 选举基本原理其实就是利用通过Kubernetes中 configmap ， endpoints 或者 lease 资源实现一个分布式锁，抢\(acqure\)到锁的节点成为leader，并且定期更新（renew）。其他进程也在不断的尝试进行抢占，抢占不到则继续等待下次循环。当leader节点挂掉之后，租约到期，其他节点就成为新的leader。



抢到锁其实就是成功把该进程的相关信息（如进程唯一标识）写入configmap、endpoints 或者 lease 资源对象中；而定期更新其实就是定期更新该资源的锁更新时间，以延续租期。



多个进程同时获取锁（更新锁资源）时，由apiserver来保证锁资源update的原子操作，通过对比resourceVersion版本号（resourceVersion的取值最终来源于etcd的modifiedindex），保证只有一个进程能修改成功，也即只有一个进程能成功获取到锁。



锁示例如下：

```
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  creationTimestamp: "2020-08-21T11:56:46Z"
  name: rbd-csi-ceph-com
  namespace: default
  resourceVersion: "69642798"
  selfLink: /apis/coordination.k8s.io/v1/namespaces/default/leases/rbd-csi-ceph-com
  uid: c9a7ea00-c000-4c5c-b90f-d0e7c85240ca
spec:
  acquireTime: "2020-08-21T11:56:46.907075Z"
  holderIdentity: node-1
  leaseDurationSeconds: 15
  leaseTransitions: 0
  renewTime: "2020-09-07T02:38:24.587170Z"

```

其中holderIdentity记录了获取到锁的进程信息，renewTime记录了锁更新时间。



**external-provisioner的Leader 选举**

从main方法代码中可以看出，在external-provisioner组件中，仅支持endpoint与lease两种锁资源，且endpoints锁会在后续被弃用，所以建议使用lease锁。



external-provisioner组件的高可用选主逻辑与k8s中的kube-controller-manager、kube-scheduler等组件的高可用选主逻辑类似。



概要过程：

（1）组件启动时，定期循环的去获取lease锁，获取成功则成为leader且返回，否则一直阻塞；

（2）获取lease锁成功，则竞选leader成功，然后运行external-provisioner组件的主体处理逻辑；

（3）竞选leader成功后，继续定期循环续约，以保证leader的长久性。



下面进行代码的分析。



**le.Run\(\)**

当--enable-leader-election组件启动参数为true时，运行该方法，主要逻辑为：

（1）定义leaderConfig结构体；

（2）调用leaderelection.RunOrDie做进一步的选主逻辑处理。

```
func (l *leaderElection) Run() error {
	if l.identity == "" {
		id, err := defaultLeaderElectionIdentity()
		if err != nil {
			return fmt.Errorf("error getting the default leader identity: %v", err)
		}

		l.identity = id
	}

	if l.namespace == "" {
		l.namespace = inClusterNamespace()
	}

	broadcaster := record.NewBroadcaster()
	broadcaster.StartRecordingToSink(&corev1.EventSinkImpl{Interface: l.clientset.CoreV1().Events(l.namespace)})
	eventRecorder := broadcaster.NewRecorder(scheme.Scheme, v1.EventSource{Component: fmt.Sprintf("%s/%s", l.lockName, string(l.identity))})

	rlConfig := resourcelock.ResourceLockConfig{
		Identity:      sanitizeName(l.identity),
		EventRecorder: eventRecorder,
	}

	lock, err := resourcelock.New(l.resourceLock, l.namespace, sanitizeName(l.lockName), l.clientset.CoreV1(), l.clientset.CoordinationV1(), rlConfig)
	if err != nil {
		return err
	}

	leaderConfig := leaderelection.LeaderElectionConfig{
		Lock:          lock,
		LeaseDuration: l.leaseDuration,
		RenewDeadline: l.renewDeadline,
		RetryPeriod:   l.retryPeriod,
		Callbacks: leaderelection.LeaderCallbacks{
			OnStartedLeading: func(ctx context.Context) {
				klog.V(2).Info("became leader, starting")
				l.runFunc(ctx)
			},
			OnStoppedLeading: func() {
				klog.Fatal("stopped leading")
			},
			OnNewLeader: func(identity string) {
				klog.V(3).Infof("new leader detected, current leader: %s", identity)
			},
		},
	}

	leaderelection.RunOrDie(context.TODO(), leaderConfig)
	return nil // should never reach here
}

```

**leaderelection.RunOrDie\(\)**

主要逻辑：

（1）调用le.acquire\(\)方法来尝试竞选为leader（acquire方法会定期循环的去获取lease锁，获取成功则成为leader且返回，否则一直阻塞）；

（2）竞选leader成功，运行run方法；

（3）调用le.renew\(\)续约方法，定期循环续约。

```
// RunOrDie starts a client with the provided config or panics if the config
// fails to validate.
func RunOrDie(ctx context.Context, lec LeaderElectionConfig) {
	le, err := NewLeaderElector(lec)
	if err != nil {
		panic(err)
	}
	if lec.WatchDog != nil {
		lec.WatchDog.SetLeaderElection(le)
	}
	le.Run(ctx)
}

// Run starts the leader election loop
func (le *LeaderElector) Run(ctx context.Context) {
	defer func() {
		runtime.HandleCrash()
		le.config.Callbacks.OnStoppedLeading()
	}()
	if !le.acquire(ctx) {
		return // ctx signalled done
	}
	ctx, cancel := context.WithCancel(ctx)
	defer cancel()
	go le.config.Callbacks.OnStartedLeading(ctx)
	le.renew(ctx)
}

// acquire会不断循环的去获取lease锁，获取成功则成为leader且返回
// acquire loops calling tryAcquireOrRenew and returns true immediately when tryAcquireOrRenew succeeds.
// Returns false if ctx signals done.
func (le *LeaderElector) acquire(ctx context.Context) bool {
	ctx, cancel := context.WithCancel(ctx)
	defer cancel()
	succeeded := false
	desc := le.config.Lock.Describe()
	klog.Infof("attempting to acquire leader lease  %v...", desc)
	wait.JitterUntil(func() {
		succeeded = le.tryAcquireOrRenew()
		le.maybeReportTransition()
		if !succeeded {
			klog.V(4).Infof("failed to acquire lease %v", desc)
			return
		}
		le.config.Lock.RecordEvent("became leader")
		le.metrics.leaderOn(le.config.Name)
		klog.Infof("successfully acquired lease %v", desc)
		cancel()
	}, le.config.RetryPeriod, JitterFactor, true, ctx.Done())
	return succeeded
}

// 续约方法，不断循环续约
// renew loops calling tryAcquireOrRenew and returns immediately when tryAcquireOrRenew fails or ctx signals done.
func (le *LeaderElector) renew(ctx context.Context) {
	ctx, cancel := context.WithCancel(ctx)
	defer cancel()
	wait.Until(func() {
		timeoutCtx, timeoutCancel := context.WithTimeout(ctx, le.config.RenewDeadline)
		defer timeoutCancel()
		err := wait.PollImmediateUntil(le.config.RetryPeriod, func() (bool, error) {
			done := make(chan bool, 1)
			go func() {
				defer close(done)
				done <- le.tryAcquireOrRenew()
			}()

			select {
			case <-timeoutCtx.Done():
				return false, fmt.Errorf("failed to tryAcquireOrRenew %s", timeoutCtx.Err())
			case result := <-done:
				return result, nil
			}
		}, timeoutCtx.Done())

		le.maybeReportTransition()
		desc := le.config.Lock.Describe()
		if err == nil {
			klog.V(5).Infof("successfully renewed lease %v", desc)
			return
		}
		le.config.Lock.RecordEvent("stopped leading")
		le.metrics.leaderOff(le.config.Name)
		klog.Infof("failed to renew lease %v: %v", desc, err)
		cancel()
	}, le.config.RetryPeriod, ctx.Done())

	// if we hold the lease, give it up
	if le.config.ReleaseOnCancel {
		le.release()
	}
}

```

**总结**

external-provisioner组件主要list/watch pvc对象的新增、更新事件，以及pv对象的新增、更新、删除事件，然后放入claimQueue与volumeQueue，接着provisionController负责处理claimQueue（，根据需要调用ceph-csi组件的CreateVolume方法来创建存储，并创建pv对象，provisionController处理volumeQueue，根据pv的状态以及回收策略决定是否调用ceph-csi组件的DeleteVolume方法来删除存储，并删除pv对象。



**Leader 选举基本原理**

在 Golang 中，k8s client-go 这个package 针对 Leader 相关功能进行了封装，支持3种锁资源，endpoint，configmap，lease，方便使用。



Leader 选举基本原理其实就是利用通过Kubernetes中 configmap ， endpoints 或者 lease 资源实现一个分布式锁，抢\(acqure\)到锁的节点成为leader，并且定期更新（renew）。其他进程也在不断的尝试进行抢占，抢占不到则继续等待下次循环。当leader节点挂掉之后，租约到期，其他节点就成为新的leader。



抢到锁其实就是成功把该进程的相关信息（如进程唯一标识）写入configmap、endpoints 或者 lease 资源对象中；而定期更新其实就是定期更新该资源的锁更新时间，以延续租期。



多个进程同时获取锁（更新锁资源）时，由apiserver来保证锁资源update的原子操作，通过对比resourceVersion版本号（resourceVersion的取值最终来源于etcd的modifiedindex），保证只有一个进程能修改成功，也即只有一个进程能成功获取到锁。



