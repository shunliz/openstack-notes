## ceph-csi源码分析（3）-rbd driver-服务入口分析

当ceph-csi组件启动时指定的driver type为rbd时，会启动rbd driver相关的服务。然后再根据controllerserver、nodeserver的参数配置，决定启动ControllerServer与IdentityServer，或NodeServer与IdentityServer。

基于tag v3.0.0

[https://github.com/ceph/ceph-csi/releases/tag/v3.0.0](https://github.com/ceph/ceph-csi/releases/tag/v3.0.0)

![](/assets/compute-container-k8s-cephcsi13633361.png)

rbd driver分析将分为4个部分，分别是服务入口分析、controllerserver分析、nodeserver分析与IdentityServer分析。

这节先进行服务入口分析。

服务入口分析

main

分析入口，根据不同的type，初始化对应的driver，并调用driver的启动方法。

```
    ......
    switch conf.Vtype {
    case rbdType:
        validateCloneDepthFlag(&conf)
        validateMaxSnaphostFlag(&conf)
        driver := rbd.NewDriver()
        driver.Run(&conf)

    case cephfsType:
        driver := cephfs.NewDriver()
        driver.Run(&conf)

    case livenessType:
        liveness.Run(&conf)

    default:
        klog.Fatalln("invalid volume type", conf.Vtype) // calls exit
    }
    ......
```

Run（rbd）

rbd driver的启动方法。

主要逻辑：

（1）创建ceph.conf文件用于执行cli命令；

（2）创建CSIDriver ，注册driver对外的服务能力，即拥有哪些功能；

（3）创建IdentityServer；

（4）根据启动参数配置，决定创建nodeserver，还是创建controllerserver；

（5）创建grpc server，并注册前面创建的几个server；

（6）启动grpc server。

```
// internal/rbd/driver.go

type Driver struct {
    cd *csicommon.CSIDriver

    ids *IdentityServer
    ns  *NodeServer
    cs  *ControllerServer
}

// Run start a non-blocking grpc controller,node and identityserver for
// rbd CSI driver which can serve multiple parallel requests.
func (r *Driver) Run(conf *util.Config) {
    var err error
    var topology map[string]string

    // 创建ceph.conf文件用于执行cli命令
    // Create ceph.conf for use with CLI commands
    if err = util.WriteCephConfig(); err != nil {
        klog.Fatalf("failed to write ceph configuration file (%v)", err)
    }

    // Use passed in instance ID, if provided for omap suffix naming
    if conf.InstanceID != "" {
        CSIInstanceID = conf.InstanceID
    }

    // update clone soft and hard limit
    rbdHardMaxCloneDepth = conf.RbdHardMaxCloneDepth
    rbdSoftMaxCloneDepth = conf.RbdSoftMaxCloneDepth
    skipForceFlatten = conf.SkipForceFlatten
    maxSnapshotsOnImage = conf.MaxSnapshotsOnImage
    // Create instances of the volume and snapshot journal
    volJournal = journal.NewCSIVolumeJournal(CSIInstanceID)
    snapJournal = journal.NewCSISnapshotJournal(CSIInstanceID)

    // 创建CSIDriver ，注册driver能力
    // Initialize default library driver
    r.cd = csicommon.NewCSIDriver(conf.DriverName, util.DriverVersion, conf.NodeID)
    if r.cd == nil {
        klog.Fatalln("Failed to initialize CSI Driver.")
    }
    if conf.IsControllerServer || !conf.IsNodeServer {
        r.cd.AddControllerServiceCapabilities([]csi.ControllerServiceCapability_RPC_Type{
            csi.ControllerServiceCapability_RPC_CREATE_DELETE_VOLUME,
            csi.ControllerServiceCapability_RPC_CREATE_DELETE_SNAPSHOT,
            csi.ControllerServiceCapability_RPC_CLONE_VOLUME,
            csi.ControllerServiceCapability_RPC_EXPAND_VOLUME,
        })
        // We only support the multi-writer option when using block, but it's a supported capability for the plugin in general
        // In addition, we want to add the remaining modes like MULTI_NODE_READER_ONLY,
        // MULTI_NODE_SINGLE_WRITER etc, but need to do some verification of RO modes first
        // will work those as follow up features
        r.cd.AddVolumeCapabilityAccessModes(
            []csi.VolumeCapability_AccessMode_Mode{csi.VolumeCapability_AccessMode_SINGLE_NODE_WRITER,
                csi.VolumeCapability_AccessMode_MULTI_NODE_MULTI_WRITER})
    }

    // 创建IdentityServer
    // Create GRPC servers
    r.ids = NewIdentityServer(r.cd)

    // 创建nodeserver
    if conf.IsNodeServer {
        topology, err = util.GetTopologyFromDomainLabels(conf.DomainLabels, conf.NodeID, conf.DriverName)
        if err != nil {
            klog.Fatalln(err)
        }
        r.ns, err = NewNodeServer(r.cd, conf.Vtype, topology)
        if err != nil {
            klog.Fatalf("failed to start node server, err %v\n", err)
        }
    }

    // 创建controllerserver
    if conf.IsControllerServer {
        r.cs = NewControllerServer(r.cd)
    }
    if !conf.IsControllerServer && !conf.IsNodeServer {
        topology, err = util.GetTopologyFromDomainLabels(conf.DomainLabels, conf.NodeID, conf.DriverName)
        if err != nil {
            klog.Fatalln(err)
        }
        r.ns, err = NewNodeServer(r.cd, conf.Vtype, topology)
        if err != nil {
            klog.Fatalf("failed to start node server, err %v\n", err)
        }
        r.cs = NewControllerServer(r.cd)
    }

    // 创建grpc server，注册前面创建的几个server并启动grpc server
    s := csicommon.NewNonBlockingGRPCServer()
    s.Start(conf.Endpoint, conf.HistogramOption, r.ids, r.cs, r.ns, conf.EnableGRPCMetrics)
    if conf.EnableGRPCMetrics {
        klog.Warning("EnableGRPCMetrics is deprecated")
        go util.StartMetricsServer(conf)
    }
    s.Wait()
}
```



