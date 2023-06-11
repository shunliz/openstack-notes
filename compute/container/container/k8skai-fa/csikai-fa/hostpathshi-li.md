## 实例 {#实例}

csi-driver-host-path 是社区实现的一个 CSI 插件的示例，它以 hostpath 为后端存储，kubernetes 通过这个 CSI 插件 driver 来对接 hostpath ，管理本地 Node 节点上的存储卷。我们以这个为例子，来看看如何编写一个csi插件。

### 目录结构 {#目录结构}

```
├── hostpath
│   ├── controllerserver.go
│   ├── nodeserver.go
│   ├── nodeserver_test.go
│   ├── hostpath.go
│   └── utils.go
│   └── identityserver.go
```

* controllerserver.go主要是实现controller的rpc接口

* nodeserver.go主要是实现node的rpc接口

* utils.go基本公用函数

* hostpath.go入口，启动rpc服务，只要把编写好的 gRPC Server 注册给 CSI，它就可以响应来自 External Components 的 CSI 请求了。
* identityserver.go身份验证的rpc接口

### 启动rpc服务 {#启动rpc服务}

首先肯定是定一个结构体包含plugin启动的所需信息

```
type hostPath struct {
    driver *csicommon.CSIDriver

    ids *identityServer
    ns  *nodeServer
    cs  *controllerServer

    cap   []*csi.VolumeCapability_AccessMode
    cscap []*csi.ControllerServiceCapability
}
```

简单的介绍一下这个结构体

* csicommon.CSIDriver：k8s自定义代表插件的结构体, 初始化的时候需要指定插件的RPC功能和支持的读写模式
* endpoint：插件的监听地址,一般的,我们测试的时候可以用tcp方式进行,比如
  `tcp://127.0.0.1:10000`
  ,最后在k8s中部署的时候一般使用unix方式:/csi/csi.sock
* csicommon.DefaultIdentityServer: 认证服务一般不需要特别实现,使用k8s公共部分的即可
* controllerServer: 实现CSI中的controller服务的RPC功能,继承后可以选择性覆盖部分方法
* nodeServer: 实现CSI中的node服务的RPC功能,继承后可以选择性覆盖部分方法

然后就是调用这个结构体的run方法，该方法中调用csicommon的公共方法启动socket监听

```
func (hp *hostPath) Run(driverName, nodeID, endpoint string) {
    glog.Infof("Driver: %v ", driverName)
    glog.Infof("Version: %s", vendorVersion)

    // Initialize default library driver
    hp.driver = csicommon.NewCSIDriver(driverName, vendorVersion, nodeID)
    if hp.driver == nil {
        glog.Fatalln("Failed to initialize CSI Driver.")
    }
    hp.driver.AddControllerServiceCapabilities(
        []csi.ControllerServiceCapability_RPC_Type{
            csi.ControllerServiceCapability_RPC_CREATE_DELETE_VOLUME,
            csi.ControllerServiceCapability_RPC_CREATE_DELETE_SNAPSHOT,
            csi.ControllerServiceCapability_RPC_LIST_SNAPSHOTS,
        })
    hp.driver.AddVolumeCapabilityAccessModes([]csi.VolumeCapability_AccessMode_Mode{csi.VolumeCapability_AccessMode_SINGLE_NODE_WRITER})

    // Create GRPC servers
    hp.ids = NewIdentityServer(hp.driver)
    hp.ns = NewNodeServer(hp.driver)
    hp.cs = NewControllerServer(hp.driver)

    s := csicommon.NewNonBlockingGRPCServer()
    s.Start(endpoint, hp.ids, hp.cs, hp.ns)
    s.Wait()
}
```

start来启动grpc服务来给对应的客户端进行调用

```
func (s *nonBlockingGRPCServer) serve(endpoint string, ids csi.IdentityServer, cs csi.ControllerServer, ns csi.NodeServer) {

    proto, addr, err := parseEndpoint(endpoint)
    if err != nil {
        glog.Fatal(err.Error())
    }

    if proto == "unix" {
        addr = "/" + addr
        if err := os.Remove(addr); err != nil && !os.IsNotExist(err) { //nolint: vetshadow
            glog.Fatalf("Failed to remove %s, error: %s", addr, err.Error())
        }
    }

    listener, err := net.Listen(proto, addr)
    if err != nil {
        glog.Fatalf("Failed to listen: %v", err)
    }

    opts := []grpc.ServerOption{
        grpc.UnaryInterceptor(logGRPC),
    }
    server := grpc.NewServer(opts...)
    s.server = server

    if ids != nil {
        csi.RegisterIdentityServer(server, ids)
    }
    if cs != nil {
        csi.RegisterControllerServer(server, cs)
    }
    if ns != nil {
        csi.RegisterNodeServer(server, ns)
    }

    glog.Infof("Listening for connections on address: %#v", listener.Addr())

    server.Serve(listener)

}
```

### 实现CSI Identity {#实现csi-identity}

然后就是对应接口的实现，先看CSI Identity 用于认证driver的身份信息，上面提到的 kubernetes 外部组件调用，返回 CSI driver 的身份信息和健康状态。

```
func NewIdentityServer(name, version string) *identityServer {
    return &identityServer{
        name:    name,
        version: version,
    }
}

func (ids *identityServer) GetPluginInfo(ctx context.Context, req *csi.GetPluginInfoRequest) (*csi.GetPluginInfoResponse, error) {
    glog.V(5).Infof("Using default GetPluginInfo")

    if ids.name == "" {
        return nil, status.Error(codes.Unavailable, "Driver name not configured")
    }

    if ids.version == "" {
        return nil, status.Error(codes.Unavailable, "Driver is missing version")
    }

    return &csi.GetPluginInfoResponse{
        Name:          ids.name,
        VendorVersion: ids.version,
    }, nil
}

func (ids *identityServer) Probe(ctx context.Context, req *csi.ProbeRequest) (*csi.ProbeResponse, error) {
    return &csi.ProbeResponse{}, nil
}

func (ids *identityServer) GetPluginCapabilities(ctx context.Context, req *csi.GetPluginCapabilitiesRequest) (*csi.GetPluginCapabilitiesResponse, error) {
    glog.V(5).Infof("Using default capabilities")
    return &csi.GetPluginCapabilitiesResponse{
        Capabilities: []*csi.PluginCapability{
            {
                Type: &csi.PluginCapability_Service_{
                    Service: &csi.PluginCapability_Service{
                        Type: csi.PluginCapability_Service_CONTROLLER_SERVICE,
                    },
                },
            },
        },
    }, nil
}
```

其中name就是我们每次在sc中声明的csi插件的值，Kubernetes 正是通过 GetPluginInfo 的返回值，来找到你在 StorageClass 里声明要使用的 CSI 插件的。

另外一个 GetPluginCapabilities 接口也很重要。这个接口返回的是这个 CSI 插件的“能力”。比如，当你编写的 CSI 插件不准备实现“Provision 阶段”和“Attach 阶段”（比如，一个最简单的 NFS 存储插件就不需要这两个阶段）时，你就可以通过这个接口返回：本插件不提供 CSI Controller 服务，即：没有 csi.PluginCapability\_Service\_CONTROLLER\_SERVICE 这个“能力”。这样，Kubernetes 就知道这个信息了。

最后，CSI Identity 服务还提供了一个 Probe 接口。Kubernetes 会调用它来检查这个 CSI 插件是否正常工作。一般情况下，我建议你在编写插件时给它设置一个 Ready 标志，当插件的 gRPC Server 停止的时候，把这个 Ready 标志设置为 false。或者，你可以在这里访问一下插件的端口，类似于健康检查的做法。

### 实现CSI Controller {#实现csi-controller}

再来看看CSI Controller 主要实现 Volume 管理流程当中的 “Provision” 和 “Attach” 阶段。”Provision” 阶段是指创建和删除 Volume 的流程，而 “Attach” 阶段是指把存储卷附着在某个 Node 或脱离某个 Node 的流程。只有块存储类型的 CSI 插件才需要 “Attach” 功能。

```
func NewControllerServer(ephemeral bool) *controllerServer {
    if ephemeral {
        return &controllerServer{caps: getControllerServiceCapabilities(nil)}
    }
    return &controllerServer{
        caps: getControllerServiceCapabilities(
            []csi.ControllerServiceCapability_RPC_Type{
                csi.ControllerServiceCapability_RPC_CREATE_DELETE_VOLUME,
                csi.ControllerServiceCapability_RPC_CREATE_DELETE_SNAPSHOT,
                csi.ControllerServiceCapability_RPC_LIST_SNAPSHOTS,
            }),
    }
}

func (cs *controllerServer) CreateVolume(ctx context.Context, req *csi.CreateVolumeRequest) (*csi.CreateVolumeResponse, error) {
    if err := cs.validateControllerServiceRequest(csi.ControllerServiceCapability_RPC_CREATE_DELETE_VOLUME); err != nil {
        glog.V(3).Infof("invalid create volume req: %v", req)
        return nil, err
    }

    // Check arguments
    if len(req.GetName()) == 0 {
        return nil, status.Error(codes.InvalidArgument, "Name missing in request")
    }
    caps := req.GetVolumeCapabilities()
    if caps == nil {
        return nil, status.Error(codes.InvalidArgument, "Volume Capabilities missing in request")
    }

    // Keep a record of the requested access types.
    var accessTypeMount, accessTypeBlock bool

    for _, cap := range caps {
        if cap.GetBlock() != nil {
            accessTypeBlock = true
        }
        if cap.GetMount() != nil {
            accessTypeMount = true
        }
    }
    // A real driver would also need to check that the other
    // fields in VolumeCapabilities are sane. The check above is
    // just enough to pass the "[Testpattern: Dynamic PV (block
    // volmode)] volumeMode should fail in binding dynamic
    // provisioned PV to PVC" storage E2E test.

    if accessTypeBlock && accessTypeMount {
        return nil, status.Error(codes.InvalidArgument, "cannot have both block and mount access type")
    }

    var requestedAccessType accessType

    if accessTypeBlock {
        requestedAccessType = blockAccess
    } else {
        // Default to mount.
        requestedAccessType = mountAccess
    }

    // Check for maximum available capacity
    capacity := int64(req.GetCapacityRange().GetRequiredBytes())
    if capacity >= maxStorageCapacity {
        return nil, status.Errorf(codes.OutOfRange, "Requested capacity %d exceeds maximum allowed %d", capacity, maxStorageCapacity)
    }

    // Need to check for already existing volume name, and if found
    // check for the requested capacity and already allocated capacity
    if exVol, err := getVolumeByName(req.GetName()); err == nil {
        // Since err is nil, it means the volume with the same name already exists
        // need to check if the size of exisiting volume is the same as in new
        // request
        if exVol.VolSize >= int64(req.GetCapacityRange().GetRequiredBytes()) {
            // exisiting volume is compatible with new request and should be reused.
            // TODO (sbezverk) Do I need to make sure that RBD volume still exists?
            return &csi.CreateVolumeResponse{
                Volume: &csi.Volume{
                    VolumeId:      exVol.VolID,
                    CapacityBytes: int64(exVol.VolSize),
                    VolumeContext: req.GetParameters(),
                },
            }, nil
        }
        return nil, status.Error(codes.AlreadyExists, fmt.Sprintf("Volume with the same name: %s but with different size already exist", req.GetName()))
    }

    volumeID := uuid.NewUUID().String()
    path := getVolumePath(volumeID)

    if requestedAccessType == blockAccess {
        executor := utilexec.New()
        size := fmt.Sprintf("%dM", capacity/mib)
        // Create a block file.
        out, err := executor.Command("fallocate", "-l", size, path).CombinedOutput()
        if err != nil {
            glog.V(3).Infof("failed to create block device: %v", string(out))
            return nil, err
        }

        // Associate block file with the loop device.
        volPathHandler := volumepathhandler.VolumePathHandler{}
        _, err = volPathHandler.AttachFileDevice(path)
        if err != nil {
            glog.Errorf("failed to attach device: %v", err)
            // Remove the block file because it'll no longer be used again.
            if err2 := os.Remove(path); err != nil {
                glog.Errorf("failed to cleanup block file %s: %v", path, err2)
            }
            return nil, status.Error(codes.Internal, fmt.Sprintf("failed to attach device: %v", err))
        }
    }

    vol, err := createHostpathVolume(volumeID, req.GetName(), capacity, requestedAccessType)
    if err != nil {
        return nil, status.Error(codes.Internal, fmt.Sprintf("failed to create volume: %s", err))
    }
    glog.V(4).Infof("created volume %s at path %s", vol.VolID, vol.VolPath)

    if req.GetVolumeContentSource() != nil {
        contentSource := req.GetVolumeContentSource()
        if contentSource.GetSnapshot() != nil {
            snapshotId := contentSource.GetSnapshot().GetSnapshotId()
            snapshot, ok := hostPathVolumeSnapshots[snapshotId]
            if !ok {
                return nil, status.Errorf(codes.NotFound, "cannot find snapshot %v", snapshotId)
            }
            if snapshot.ReadyToUse != true {
                return nil, status.Errorf(codes.Internal, "Snapshot %v is not yet ready to use.", snapshotId)
            }
            snapshotPath := snapshot.Path
            args := []string{"zxvf", snapshotPath, "-C", path}
            executor := utilexec.New()
            out, err := executor.Command("tar", args...).CombinedOutput()
            if err != nil {
                return nil, status.Error(codes.Internal, fmt.Sprintf("failed pre-populate data for volume: %v: %s", err, out))
            }
        }
    }

    return &csi.CreateVolumeResponse{
        Volume: &csi.Volume{
            VolumeId:      volumeID,
            CapacityBytes: req.GetCapacityRange().GetRequiredBytes(),
            VolumeContext: req.GetParameters(),
        },
    }, nil
}

func (cs *controllerServer) DeleteVolume(ctx context.Context, req *csi.DeleteVolumeRequest) (*csi.DeleteVolumeResponse, error) {

    // Check arguments
    if len(req.GetVolumeId()) == 0 {
        return nil, status.Error(codes.InvalidArgument, "Volume ID missing in request")
    }

    if err := cs.validateControllerServiceRequest(csi.ControllerServiceCapability_RPC_CREATE_DELETE_VOLUME); err != nil {
        glog.V(3).Infof("invalid delete volume req: %v", req)
        return nil, err
    }

    vol, err := getVolumeByID(req.GetVolumeId())
    if err != nil {
        // Return OK if the volume is not found.
        return &csi.DeleteVolumeResponse{}, nil
    }
    glog.V(4).Infof("deleting volume %s", vol.VolID)

    if vol.VolAccessType == blockAccess {

        volPathHandler := volumepathhandler.VolumePathHandler{}
        // Get the associated loop device.
        device, err := volPathHandler.GetLoopDevice(getVolumePath(vol.VolID))
        if err != nil {
            return nil, status.Error(codes.Internal, fmt.Sprintf("failed to get the loop device: %v", err))
        }

        if device != "" {
            // Remove any associated loop device.
            glog.V(4).Infof("deleting loop device %s", device)
            if err := volPathHandler.RemoveLoopDevice(device); err != nil {
                return nil, status.Error(codes.Internal, fmt.Sprintf("failed to remove loop device: %v", err))
            }
        }
    }

    if err := deleteHostpathVolume(vol.VolID); err != nil && !os.IsNotExist(err) {
        return nil, status.Error(codes.Internal, fmt.Sprintf("failed to delete volume: %s", err))
    }

    glog.V(4).Infof("volume deleted ok: %s", vol.VolID)

    return &csi.DeleteVolumeResponse{}, nil
}
```

其中，CreateVolume 和 DeleteVolume 是实现 “Provision” 阶段需要实现的接口，External provisioner 组件会 CSI 插件的这个接口以创建或者删除存储卷。

CreateVolume其实就是调用对应存储的api来创建一个存储卷，如果你使用的是其他类型的块存储（比如 Cinder、Ceph RBD 等），对应的操作也是类似地调用创建存储卷的 API。

ControllerPublishVolume 和 ControllerUnpublishVolume 是实现 “Attach” 阶段需要实现的接口，External attach 组件会调用 CSI 插件实现的这个接口以把某个块存储卷附着或脱离某个 Node 。

ControllerPublishVolume其实就是调用对应的存储的api将某个块存储卷附着某个 Node。

如果想扩展 CSI 的功能，可以实现更多功能的接口，如快照功能的接口 CreateSnapshot 和 DeleteSnapshot。

### 实现CSI Node {#实现csi-node}

最后再来看看CSI Node 部分主要负责 Volume 管理流程当中的 “Mount” 阶段，即把 Volume 挂载至 Pod 容器，或者从 Pod 中卸载 Volume 。在宿主机 Node 上需要执行的操作都包含在这个部分。

```
func NewNodeServer(nodeId string, ephemeral bool) *nodeServer {
    return &nodeServer{
        nodeID:    nodeId,
        ephemeral: ephemeral,
    }
}

func (ns *nodeServer) NodePublishVolume(ctx context.Context, req *csi.NodePublishVolumeRequest) (*csi.NodePublishVolumeResponse, error) {

    // Check arguments
    if req.GetVolumeCapability() == nil {
        return nil, status.Error(codes.InvalidArgument, "Volume capability missing in request")
    }
    if len(req.GetVolumeId()) == 0 {
        return nil, status.Error(codes.InvalidArgument, "Volume ID missing in request")
    }
    if len(req.GetTargetPath()) == 0 {
        return nil, status.Error(codes.InvalidArgument, "Target path missing in request")
    }

    targetPath := req.GetTargetPath()

    if req.GetVolumeCapability().GetBlock() != nil &&
        req.GetVolumeCapability().GetMount() != nil {
        return nil, status.Error(codes.InvalidArgument, "cannot have both block and mount access type")
    }

    // if ephemeral is specified, create volume here to avoid errors
    if ns.ephemeral {
        volID := req.GetVolumeId()
        volName := fmt.Sprintf("ephemeral-%s", volID)
        vol, err := createHostpathVolume(req.GetVolumeId(), volName, maxStorageCapacity, mountAccess)
        if err != nil && !os.IsExist(err) {
            glog.Error("ephemeral mode failed to create volume: ", err)
            return nil, status.Error(codes.Internal, err.Error())
        }
        glog.V(4).Infof("ephemeral mode: created volume: %s", vol.VolPath)
    }

    vol, err := getVolumeByID(req.GetVolumeId())
    if err != nil {
        return nil, status.Error(codes.NotFound, err.Error())
    }

    if req.GetVolumeCapability().GetBlock() != nil {
        if vol.VolAccessType != blockAccess {
            return nil, status.Error(codes.InvalidArgument, "cannot publish a non-block volume as block volume")
        }

        volPathHandler := volumepathhandler.VolumePathHandler{}

        // Get loop device from the volume path.
        loopDevice, err := volPathHandler.GetLoopDevice(vol.VolPath)
        if err != nil {
            return nil, status.Error(codes.Internal, fmt.Sprintf("failed to get the loop device: %v", err))
        }

        mounter := mount.New("")

        // Check if the target path exists. Create if not present.
        _, err = os.Lstat(targetPath)
        if os.IsNotExist(err) {
            if err = mounter.MakeFile(targetPath); err != nil {
                return nil, status.Error(codes.Internal, fmt.Sprintf("failed to create target path: %s: %v", targetPath, err))
            }
        }
        if err != nil {
            return nil, status.Errorf(codes.Internal, "failed to check if the target block file exists: %v", err)
        }

        // Check if the target path is already mounted. Prevent remounting.
        notMount, err := mounter.IsNotMountPoint(targetPath)
        if err != nil {
            if !os.IsNotExist(err) {
                return nil, status.Errorf(codes.Internal, "error checking path %s for mount: %s", targetPath, err)
            }
            notMount = true
        }
        if !notMount {
            // It's already mounted.
            glog.V(5).Infof("Skipping bind-mounting subpath %s: already mounted", targetPath)
            return &csi.NodePublishVolumeResponse{}, nil
        }

        options := []string{"bind"}
        if err := mount.New("").Mount(loopDevice, targetPath, "", options); err != nil {
            return nil, status.Error(codes.Internal, fmt.Sprintf("failed to mount block device: %s at %s: %v", loopDevice, targetPath, err))
        }
    } else if req.GetVolumeCapability().GetMount() != nil {
        if vol.VolAccessType != mountAccess {
            return nil, status.Error(codes.InvalidArgument, "cannot publish a non-mount volume as mount volume")
        }

        notMnt, err := mount.New("").IsLikelyNotMountPoint(targetPath)
        if err != nil {
            if os.IsNotExist(err) {
                if err = os.MkdirAll(targetPath, 0750); err != nil {
                    return nil, status.Error(codes.Internal, err.Error())
                }
                notMnt = true
            } else {
                return nil, status.Error(codes.Internal, err.Error())
            }
        }

        if !notMnt {
            return &csi.NodePublishVolumeResponse{}, nil
        }

        fsType := req.GetVolumeCapability().GetMount().GetFsType()

        deviceId := ""
        if req.GetPublishContext() != nil {
            deviceId = req.GetPublishContext()[deviceID]
        }

        readOnly := req.GetReadonly()
        volumeId := req.GetVolumeId()
        attrib := req.GetVolumeContext()
        mountFlags := req.GetVolumeCapability().GetMount().GetMountFlags()

        glog.V(4).Infof("target %v\nfstype %v\ndevice %v\nreadonly %v\nvolumeId %v\nattributes %v\nmountflags %v\n",
            targetPath, fsType, deviceId, readOnly, volumeId, attrib, mountFlags)

        options := []string{"bind"}
        if readOnly {
            options = append(options, "ro")
        }
        mounter := mount.New("")
        path := getVolumePath(volumeId)

        if err := mounter.Mount(path, targetPath, "", options); err != nil {
            var errList strings.Builder
            errList.WriteString(err.Error())
            if ns.ephemeral {
                if rmErr := os.RemoveAll(path); rmErr != nil && !os.IsNotExist(rmErr) {
                    errList.WriteString(fmt.Sprintf(" :%s", rmErr.Error()))
                }
            }
        }
    }

    return &csi.NodePublishVolumeResponse{}, nil
}

func (ns *nodeServer) NodeUnpublishVolume(ctx context.Context, req *csi.NodeUnpublishVolumeRequest) (*csi.NodeUnpublishVolumeResponse, error) {

    // Check arguments
    if len(req.GetVolumeId()) == 0 {
        return nil, status.Error(codes.InvalidArgument, "Volume ID missing in request")
    }
    if len(req.GetTargetPath()) == 0 {
        return nil, status.Error(codes.InvalidArgument, "Target path missing in request")
    }
    targetPath := req.GetTargetPath()
    volumeID := req.GetVolumeId()

    vol, err := getVolumeByID(volumeID)
    if err != nil {
        return nil, status.Error(codes.NotFound, err.Error())
    }

    switch vol.VolAccessType {
    case blockAccess:
        // Unmount and delete the block file.
        err = mount.New("").Unmount(targetPath)
        if err != nil {
            return nil, status.Error(codes.Internal, err.Error())
        }
        if err = os.RemoveAll(targetPath); err != nil {
            return nil, status.Error(codes.Internal, err.Error())
        }
        glog.V(4).Infof("hostpath: volume %s has been unpublished.", targetPath)
    case mountAccess:
        // Unmounting the image
        err = mount.New("").Unmount(req.GetTargetPath())
        if err != nil {
            return nil, status.Error(codes.Internal, err.Error())
        }
        glog.V(4).Infof("hostpath: volume %s/%s has been unmounted.", targetPath, volumeID)
    }

    if ns.ephemeral {
        glog.V(4).Infof("deleting volume %s", volumeID)
        if err := deleteHostpathVolume(volumeID); err != nil && !os.IsNotExist(err) {
            return nil, status.Error(codes.Internal, fmt.Sprintf("failed to delete volume: %s", err))
        }
    }

    return &csi.NodeUnpublishVolumeResponse{}, nil
}

func (ns *nodeServer) NodeStageVolume(ctx context.Context, req *csi.NodeStageVolumeRequest) (*csi.NodeStageVolumeResponse, error) {

    // Check arguments
    if len(req.GetVolumeId()) == 0 {
        return nil, status.Error(codes.InvalidArgument, "Volume ID missing in request")
    }
    if len(req.GetStagingTargetPath()) == 0 {
        return nil, status.Error(codes.InvalidArgument, "Target path missing in request")
    }
    if req.GetVolumeCapability() == nil {
        return nil, status.Error(codes.InvalidArgument, "Volume Capability missing in request")
    }

    return &csi.NodeStageVolumeResponse{}, nil
}

func (ns *nodeServer) NodeUnstageVolume(ctx context.Context, req *csi.NodeUnstageVolumeRequest) (*csi.NodeUnstageVolumeResponse, error) {

    // Check arguments
    if len(req.GetVolumeId()) == 0 {
        return nil, status.Error(codes.InvalidArgument, "Volume ID missing in request")
    }
    if len(req.GetStagingTargetPath()) == 0 {
        return nil, status.Error(codes.InvalidArgument, "Target path missing in request")
    }

    return &csi.NodeUnstageVolumeResponse{}, nil
}
```

kubelet 会调用 CSI 插件实现的接口，以实现 volume 的挂载和卸载。

其中 Volume 的挂载被分成了 NodeStageVolume 和 NodePublishVolume 两个阶段。NodeStageVolume 接口主要是针对块存储类型的 CSI 插件而提供的。 块设备在 “Attach” 阶段被附着在 Node 上后，需要挂载至 Pod 对应目录上，但因为块设备在 linux 上只能 mount 一次，而在 kubernetes volume 的使用场景中，一个 volume 可能被挂载进同一个 Node 上的多个 Pod 实例中，所以这里提供了 NodeStageVolume 这个接口，使用这个接口把块设备格式化后先挂载至 Node 上的一个临时全局目录，然后再调用 NodePublishVolume 使用 linux 中的 bind mount 技术把这个全局目录挂载进 Pod 中对应的目录上。

其实就是我们首先需要格式化这个设备，然后才能把它挂载到 Volume 对应的宿主机目录上，所以需要NodeStageVolume。

NodeUnstageVolume 和 NodeUnpublishVolume 正是 volume 卸载阶段所分别对应的上述两个流程。从上述代码中可以看到，因为 hostpath 非块存储类型的第三方存储，所以没有实现 NodeStageVolume 和 NodeUnstageVolume 这两个接口。

当然，如果是非块存储类型的 CSI 插件，也就不必实现 NodeStageVolume 和 NodeUnstageVolume 这两个接口了。

到这里一个完整的csi插件就开发完成，大体都是这些流程，只要实现对应的rpc接口逻辑就好。



参考：

https://kingjcy.github.io/post/cloud/paas/base/kubernetes/k8s-store-csi/

https://blog.hdls.me/16672085188369.html



