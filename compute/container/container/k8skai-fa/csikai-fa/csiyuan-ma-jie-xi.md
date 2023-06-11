# 源码解析

## pv controller {#pv-controller}

根据存储流程，首先调度后，pv控制器来处理pv和pvc的关系，pv控制器在组件kube-controller-manager中，我们先来看看pv控制器，首先看到pv控制器在kube-controller-manager的注册。

```
controllers["persistentvolume-binder"] = startPersistentVolumeBinderController
controllers["persistentvolume-expander"] = startVolumeExpandController
```

分别支持原生的存储和扩展的存储，我们来看对待的初始化函数。

```
func startPersistentVolumeBinderController(ctx ControllerContext) (http.Handler, bool, error) {
    plugins, err := ProbeControllerVolumePlugins(ctx.Cloud, ctx.ComponentConfig.PersistentVolumeBinderController.VolumeConfiguration)
    if err != nil {
        return nil, true, fmt.Errorf("failed to probe volume plugins when starting persistentvolume controller: %v", err)
    }
    filteredDialOptions, err := options.ParseVolumeHostFilters(
        ctx.ComponentConfig.PersistentVolumeBinderController.VolumeHostCIDRDenylist,
        ctx.ComponentConfig.PersistentVolumeBinderController.VolumeHostAllowLocalLoopback)
    if err != nil {
        return nil, true, err
    }
    params := persistentvolumecontroller.ControllerParameters{
        KubeClient:                ctx.ClientBuilder.ClientOrDie("persistent-volume-binder"),
        SyncPeriod:                ctx.ComponentConfig.PersistentVolumeBinderController.PVClaimBinderSyncPeriod.Duration,
        VolumePlugins:             plugins,
        Cloud:                     ctx.Cloud,
        ClusterName:               ctx.ComponentConfig.KubeCloudShared.ClusterName,
        VolumeInformer:            ctx.InformerFactory.Core().V1().PersistentVolumes(),
        ClaimInformer:             ctx.InformerFactory.Core().V1().PersistentVolumeClaims(),
        ClassInformer:             ctx.InformerFactory.Storage().V1().StorageClasses(),
        PodInformer:               ctx.InformerFactory.Core().V1().Pods(),
        NodeInformer:              ctx.InformerFactory.Core().V1().Nodes(),
        EnableDynamicProvisioning: ctx.ComponentConfig.PersistentVolumeBinderController.VolumeConfiguration.EnableDynamicProvisioning,
        FilteredDialOptions:       filteredDialOptions,
    }
    volumeController, volumeControllerErr := persistentvolumecontroller.NewController(params)
    if volumeControllerErr != nil {
        return nil, true, fmt.Errorf("failed to construct persistentvolume controller: %v", volumeControllerErr)
    }
    go volumeController.Run(ctx.Stop)
    return nil, true, nil
}
```

创建各种params最后创建结构体PersistentVolumeController

```
// NewController creates a new PersistentVolume controller
func NewController(p ControllerParameters) (*PersistentVolumeController, error) {
    eventRecorder := p.EventRecorder
    if eventRecorder == nil {
        broadcaster := record.NewBroadcaster()
        broadcaster.StartStructuredLogging(0)
        broadcaster.StartRecordingToSink(&v1core.EventSinkImpl{Interface: p.KubeClient.CoreV1().Events("")})
        eventRecorder = broadcaster.NewRecorder(scheme.Scheme, v1.EventSource{Component: "persistentvolume-controller"})
    }

    controller := &PersistentVolumeController{
        volumes:                       newPersistentVolumeOrderedIndex(),
        claims:                        cache.NewStore(cache.DeletionHandlingMetaNamespaceKeyFunc),
        kubeClient:                    p.KubeClient,
        eventRecorder:                 eventRecorder,
        runningOperations:             goroutinemap.NewGoRoutineMap(true /* exponentialBackOffOnError */),
        cloud:                         p.Cloud,
        enableDynamicProvisioning:     p.EnableDynamicProvisioning,
        clusterName:                   p.ClusterName,
        createProvisionedPVRetryCount: createProvisionedPVRetryCount,
        createProvisionedPVInterval:   createProvisionedPVInterval,
        claimQueue:                    workqueue.NewNamed("claims"),
        volumeQueue:                   workqueue.NewNamed("volumes"),
        resyncPeriod:                  p.SyncPeriod,
        operationTimestamps:           metrics.NewOperationStartTimeCache(),
    }
    ...
}
```

然后初始化volumes的插件，包括hostpath nfs csi等等

```
// Prober is nil because PV is not aware of Flexvolume.
if err := controller.volumePluginMgr.InitPlugins(p.VolumePlugins, nil /* prober */, controller); err != nil {
    return nil, fmt.Errorf("Could not initialize volume plugins for PersistentVolume Controller: %v", err)
}

```

然后添加volume informer机制

```
p.VolumeInformer.Informer().AddEventHandler(
    cache.ResourceEventHandlerFuncs{
        AddFunc:    func(obj interface{}) { controller.enqueueWork(controller.volumeQueue, obj) },
        UpdateFunc: func(oldObj, newObj interface{}) { controller.enqueueWork(controller.volumeQueue, newObj) },
        DeleteFunc: func(obj interface{}) { controller.enqueueWork(controller.volumeQueue, obj) },
    },
)
controller.volumeLister = p.VolumeInformer.Lister()
controller.volumeListerSynced = p.VolumeInformer.Informer().HasSynced
```

然后添加claim informer机制

```
p.ClaimInformer.Informer().AddEventHandler(
    cache.ResourceEventHandlerFuncs{
        AddFunc:    func(obj interface{}) { controller.enqueueWork(controller.claimQueue, obj) },
        UpdateFunc: func(oldObj, newObj interface{}) { controller.enqueueWork(controller.claimQueue, newObj) },
        DeleteFunc: func(obj interface{}) { controller.enqueueWork(controller.claimQueue, obj) },
    },
)
controller.claimLister = p.ClaimInformer.Lister()
controller.claimListerSynced = p.ClaimInformer.Informer().HasSynced
```

最后添加 storageclas pod node资源 informer机制

```
controller.classLister = p.ClassInformer.Lister()
controller.classListerSynced = p.ClassInformer.Informer().HasSynced
controller.podLister = p.PodInformer.Lister()
controller.podIndexer = p.PodInformer.Informer().GetIndexer()
controller.podListerSynced = p.PodInformer.Informer().HasSynced
controller.NodeLister = p.NodeInformer.Lister()
controller.NodeListerSynced = p.NodeInformer.Informer().HasSynced

```

到此NewController调用就结束了，下面调用这个PersistentVolumeController的run函数运行

```
// Run starts all of this controller's control loops
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

    metrics.Register(ctrl.volumes.store, ctrl.claims, &ctrl.volumePluginMgr)

    <-stopCh
}
```

开启了三个goroutine来定期执行对应的函数

* resysc 定期list pv pvc并加入到队列
* volumeManager 这个我们在之前说过，主要是管理pv的状态迁移
* claimWorker 这个我们在之前说过，主要处理 PVC 的 add / update / delete 相关事件以及 PVC 的状态迁移

我们来看看对应的函数，先看resysc

```
// resync supplements short resync period of shared informers - we don't want
// all consumers of PV/PVC shared informer to have a short resync period,
// therefore we do our own.
func (ctrl *PersistentVolumeController) resync() {
    klog.V(4).Infof("resyncing PV controller")

    pvcs, err := ctrl.claimLister.List(labels.NewSelector())
    if err != nil {
        klog.Warningf("cannot list claims: %s", err)
        return
    }
    for _, pvc := range pvcs {
        ctrl.enqueueWork(ctrl.claimQueue, pvc)
    }

    pvs, err := ctrl.volumeLister.List(labels.NewSelector())
    if err != nil {
        klog.Warningf("cannot list persistent volumes: %s", err)
        return
    }
    for _, pv := range pvs {
        ctrl.enqueueWork(ctrl.volumeQueue, pv)
    }
}
```

可见就是获取pvcs，pvs，最后放到对应的队列中处理。我们再来看看volumeManager

```
// volumeWorker processes items from volumeQueue. It must run only once,
// syncVolume is not assured to be reentrant.
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

从队列取进行处理，未有就退出等待下一个周期在处理，如果有则 updateVolume 更新操作，可能包括 add update sync 等操作，处理并删除队列，具体处理的逻辑在上面已经说过，我们简单看一下

## extrenal provisioner\[todo\] {#extrenal-provisioner-todo}

## csi插件\[todo\] {#csi插件-todo}

## AD controller {#ad-controller}

AD Controller中核心部件包括两个缓存，一个populator，一个reconciler，一个status updater。

两个缓存分别是desired status和actual status，分别代表跟attach/detach相关对象模型的期望状态和实际状态。 reconciler通过定期比较期望状态和实际状态来决定是执行attach，还是detach，还是什么都不做。 populator负责定期从API Server同步相关模型值到期望状态缓存中。 status updater负责node的状态刷新，这里主要是当有卷做完attach/detach操作后，从记录实际状态缓存中获取每个node已经attach的卷信息，向API Server同步node.Status.VolumesAttached，通过这个状态Kubelet才知道AD Controller是否已经完成attach操作。详细代码查看这里，结构见下图：

核心的逻辑部件有一定了解之后，再梳理下它的逻辑。对于AD Controller来讲，它的核心职责就是当API Server中，有卷声明的pod与node间的关系发生变化时，需要决定是通过调用存储插件将这个pod关联的卷attach到对应node的主机（或者虚拟机）上，还是将卷从node上detach掉，这里的关系变化可能是pod调度某个node上，也可能是某个pod在node上终止掉了，所以它需要做这几件事情：

监控API Server 中的pod和node。 监控当前各个node上卷的实际attach状态。 通过对比API Server中pod和node的状态变化以及node上的实际attach状态来决定做attach、detach操作。 调用相应的卷插件执行attach，detach操作。 操作完成后，通知其它关联组件（这里主要是kubelet）做相应的业务操作。

## external-attacher\[todo\] {#external-attacher-todo}

## kubelet\[todo\] {#kubelet-todo}

# 开发csi插件 {#开发csi插件}

## 基本概念 {#基本概念}

csi插件的实现，官方已经封装好的lib，我们只要实现对应的接口就可以了，目前我们可以实现的插件有两种类型，如下：

* Controller Plugin，负责存储对象（Volume）的生命周期管理，在集群中仅需要有一个即可；
* Node Plugin，在必要时与使用 Volume 的容器所在的节点交互，提供诸如节点上的 Volume 挂载/卸载等动作支持，如有需要则在每个服务节点上均部署。

官方提供了rpc接口实现上面两个插件，如下：

* 身份服务：Node Plugin和Controller Plugin都必须实现这些RPC集。
  * GetPluginInfo， 获取 Plugin 基本信息
  * GetPluginCapabilities，获取 Plugin 支持的能力
  * Probe，探测 Plugin 的健康状态
* 控制器服务：Controller Plugin必须实现这些RPC集。CSI Controller 服务里定义的这些操作有个共同特点，那就是它们都无需在宿主机上进行，而是属于 Kubernetes 里 Volume Controller 的逻辑，也就是属于 Master 节点的一部分。需要注意的是，正如我在前面提到的那样，CSI Controller 服务的实际调用者，并不是 Kubernetes（即：通过 pkg/volume/csi 发起 CSI 请求），而是 External Provisioner 和 External Attacher。这两个 External Components，分别通过监听 PVC 和 VolumeAttachement 对象，来跟 Kubernetes 进行协作。
  * Volume CRUD，包括了扩容和容量探测等 Volume 状态检查与操作接口
  * Controller Publish/Unpublish Volume ，也就是对 CSI Volume 进行 Attach/Dettach，还包括Node 对 Volume 的访问权限管理
  * Snapshot CRD，快照的创建和删除操作，目前 CSI 定义的 Snapshot 仅用于创建 Volume，未提供回滚的语义
* 节点服务：Node Plugin必须实现这些RPC集。CSI Volume 需要在宿主机上执行的操作，都定义在了 CSI Node 服务里面
  * Node Stage/Unstage/Publish/Unpublish/GetStats Volume，节点上 Volume 的连接状态管理，也就是mount是由 NodeStageVolume 和 NodePublishVolume 两个接口共同实现的。
  * Node Expand Volume, 节点上的 Volume 扩容操作，在 volume 逻辑大小扩容之后，可能还需要同步的扩容 Volume 之上的文件系统并让使用 Volume 的 Container 感知到，所以在 Node Plugin 上需要有对应的接口
  * Node Get Capabilities/Info， Plugin 的基础属性与 Node 的属性查询

CSI 插件的三部分 CSI Identity , CSI Controller , CSI Node 可放在同一个二进制程序中实现。就是我们常见的插件，当然针对不同的存储，可以只实现一种类型就可以。

对应的rpc定义可以看源码，简单的看一下对应的定义

```
service Identity {
  rpc GetPluginInfo(GetPluginInfoRequest)
    returns (GetPluginInfoResponse) {}

  rpc GetPluginCapabilities(GetPluginCapabilitiesRequest)
    returns (GetPluginCapabilitiesResponse) {}

  rpc Probe (ProbeRequest)
    returns (ProbeResponse) {}
}

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

这些接口都定义在[csi的spec的lib](https://github.com/container-storage-interface/spec/blob/master/lib/go/csi/csi.pb.go)中，所以我们开发的时候会引用这个库，我们只要实现这些接口就可以实现csi的基本插件功能。

我们在来看一下我们说的组件和这个rpc是怎么个关系

