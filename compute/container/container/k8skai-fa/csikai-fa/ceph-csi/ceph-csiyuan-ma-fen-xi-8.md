ceph-csi源码分析（5）-rbd driver-nodeserver分析（上）

当ceph-csi组件启动时指定的driver type为rbd时，会启动rbd driver相关的服务。然后再根据controllerserver、nodeserver的参数配置，决定启动ControllerServer与IdentityServer，或NodeServer与IdentityServer。

基于tag v3.0.0

[https://github.com/ceph/ceph-csi/releases/tag/v3.0.0](https://github.com/ceph/ceph-csi/releases/tag/v3.0.0)

rbd driver分析将分为4个部分，分别是服务入口分析、controllerserver分析、nodeserver分析与IdentityServer分析。

![](/assets/compute-container-k8s-cephcsi13633361.png)这节进行nodeserver分析，nodeserver主要包括了NodeGetCapabilities（获取driver能力）、NodeGetVolumeStats（存储探测及metrics获取）、NodeStageVolume（map rbd与mount stagingPath）、NodePublishVolume（mount targetPath）、NodeUnpublishVolume（umount targetPath）、NodeUnstageVolume（umount stagingPath与unmap rbd）、NodeExpandVolume（node端存储扩容）操作，将一一进行分析。这节进行NodeGetCapabilities（获取driver能力）、NodeGetVolumeStats（存储探测及metrics获取）、NodeExpandVolume（node端存储扩容）的分析。



nodeserver分析

（1）NodeGetCapabilities

简介

NodeGetCapabilities主要用于获取该ceph-csi driver的能力。



该方法由由kubelet调用，在kubelet调用NodeExpandVolume、NodeStageVolume、NodeUnstageVolume等方法前，会先调用NodeGetCapabilities来获取该ceph-csi driver的能力，看是否支持对这些方法的调用。



kubelet中调用的有关代码位于pkg/volume/csi/csi\_client.go。



NodeGetCapabilities

NodeGetCapabilities方法中注册了csi driver的能力。



如下代码表示该csi组件支持的能力有：

（1）挂载存储到节点，把存储从节点上解除挂载；

（2）获取节点上的存储状态；

（3）存储扩容。

```
// ceph-csi/internal/rbd/nodeserver.go

// NodeGetCapabilities returns the supported capabilities of the node server.
func (ns *NodeServer) NodeGetCapabilities(ctx context.Context, req *csi.NodeGetCapabilitiesRequest) (*csi.NodeGetCapabilitiesResponse, error) {
	return &csi.NodeGetCapabilitiesResponse{
		Capabilities: []*csi.NodeServiceCapability{
			{
				Type: &csi.NodeServiceCapability_Rpc{
					Rpc: &csi.NodeServiceCapability_RPC{
						Type: csi.NodeServiceCapability_RPC_STAGE_UNSTAGE_VOLUME,
					},
				},
			},
			{
				Type: &csi.NodeServiceCapability_Rpc{
					Rpc: &csi.NodeServiceCapability_RPC{
						Type: csi.NodeServiceCapability_RPC_GET_VOLUME_STATS,
					},
				},
			},
			{
				Type: &csi.NodeServiceCapability_Rpc{
					Rpc: &csi.NodeServiceCapability_RPC{
						Type: csi.NodeServiceCapability_RPC_EXPAND_VOLUME,
					},
				},
			},
		},
	}, nil
}


```

**（2）NodeGetVolumeStats**

简介

NodeGetVolumeStats用于探测挂载存储的状态，并返回该存储的相关metrics给kubelet。



由kubelet定时循环调用，获取volume相关指标。kubelet定时调用的代码位于pkg/kubelet/server/stats/volume\_stat\_calculator.go-StartOnce\(\)。



**NodeGetVolumeStats**

主要逻辑：

（1）获取存储挂载路径；

（2）检测存储挂载路径是否为挂载点（对比指定路径与其父目录的stat结果中的device的值，如果device值不一致，则是挂载点）；

（3）通过stat获取存储挂载路径的Metrics并返回。

    // internal/csi-common/nodeserver-default.go

    // NodeGetVolumeStats returns volume stats.
    func (ns *DefaultNodeServer) NodeGetVolumeStats(ctx context.Context, req *csi.NodeGetVolumeStatsRequest) (*csi.NodeGetVolumeStatsResponse, error) {
    	// 获取存储挂载路径
    	var err error
    	targetPath := req.GetVolumePath()
    	if targetPath == "" {
    		err = fmt.Errorf("targetpath %v is empty", targetPath)
    		return nil, status.Error(codes.InvalidArgument, err.Error())
    	}
    	/*
    		volID := req.GetVolumeId()

    		TODO: Map the volumeID to the targetpath.

    		CephFS:
    		   we need secret to connect to the ceph cluster to get the volumeID from volume
    		   Name, however `secret` field/option is not available  in NodeGetVolumeStats spec,
    		   Below issue covers this request and once its available, we can do the validation
    		   as per the spec.

    		   https://github.com/container-storage-interface/spec/issues/371

    		RBD:
    		   Below issue covers this request for RBD and once its available, we can do the validation
    		   as per the spec.

    		   https://github.com/ceph/ceph-csi/issues/511

    	*/

        // 检测存储挂载路径是否为mountpoint
    	isMnt, err := util.IsMountPoint(targetPath)

    	if err != nil {
    		if os.IsNotExist(err) {
    			return nil, status.Errorf(codes.InvalidArgument, "targetpath %s doesnot exist", targetPath)
    		}
    		return nil, err
    	}
    	if !isMnt {
    		return nil, status.Errorf(codes.InvalidArgument, "targetpath %s is not mounted", targetPath)
    	}

        // 通过stat获取存储挂载路径的Metrics
    	cephMetricsProvider := volume.NewMetricsStatFS(targetPath)
    	volMetrics, volMetErr := cephMetricsProvider.GetMetrics()
    	if volMetErr != nil {
    		return nil, status.Error(codes.Internal, volMetErr.Error())
    	}

    	available, ok := (*(volMetrics.Available)).AsInt64()
    	if !ok {
    		klog.Errorf(util.Log(ctx, "failed to fetch available bytes"))
    	}
    	capacity, ok := (*(volMetrics.Capacity)).AsInt64()
    	if !ok {
    		klog.Errorf(util.Log(ctx, "failed to fetch capacity bytes"))
    		return nil, status.Error(codes.Unknown, "failed to fetch capacity bytes")
    	}
    	used, ok := (*(volMetrics.Used)).AsInt64()
    	if !ok {
    		klog.Errorf(util.Log(ctx, "failed to fetch used bytes"))
    	}
    	inodes, ok := (*(volMetrics.Inodes)).AsInt64()
    	if !ok {
    		klog.Errorf(util.Log(ctx, "failed to fetch available inodes"))
    		return nil, status.Error(codes.Unknown, "failed to fetch available inodes")
    	}
    	inodesFree, ok := (*(volMetrics.InodesFree)).AsInt64()
    	if !ok {
    		klog.Errorf(util.Log(ctx, "failed to fetch free inodes"))
    	}

    	inodesUsed, ok := (*(volMetrics.InodesUsed)).AsInt64()
    	if !ok {
    		klog.Errorf(util.Log(ctx, "failed to fetch used inodes"))
    	}
    	return &csi.NodeGetVolumeStatsResponse{
    		Usage: []*csi.VolumeUsage{
    			{
    				Available: available,
    				Total:     capacity,
    				Used:      used,
    				Unit:      csi.VolumeUsage_BYTES,
    			},
    			{
    				Available: inodesFree,
    				Total:     inodes,
    				Used:      inodesUsed,
    				Unit:      csi.VolumeUsage_INODES,
    			},
    		},
    	}, nil
    }


###### IsMountPoint

通过调用IsLikelyNotMountPoint来判断该路径是否为挂载点。

```
// internal/util/util.go

// IsMountPoint checks if the given path is mountpoint or not.
func IsMountPoint(p string) (bool, error) {
	dummyMount := mount.New("")
	notMnt, err := dummyMount.IsLikelyNotMountPoint(p)
	if err != nil {
		return false, status.Error(codes.Internal, err.Error())
	}

	return !notMnt, nil
}

```

dummyMount.IsLikelyNotMountPoint\(\)主要逻辑：

（1）对指定路径执行stat操作；

（2）对指定路径的父目录执行stat操作；

（3）通过对比指定路径与其父目录的stat结果中的device的值，判断出该路径是否为挂载点（如果device值不一致，则是挂载点）。

```
// vendor/k8s.io/utils/mount/mount_linux.go

// IsLikelyNotMountPoint determines if a directory is not a mountpoint.
// It is fast but not necessarily ALWAYS correct. If the path is in fact
// a bind mount from one part of a mount to another it will not be detected.
// It also can not distinguish between mountpoints and symbolic links.
// mkdir /tmp/a /tmp/b; mount --bind /tmp/a /tmp/b; IsLikelyNotMountPoint("/tmp/b")
// will return true. When in fact /tmp/b is a mount point. If this situation
// is of interest to you, don't use this function...
func (mounter *Mounter) IsLikelyNotMountPoint(file string) (bool, error) {
	stat, err := os.Stat(file)
	if err != nil {
		return true, err
	}
	rootStat, err := os.Stat(filepath.Dir(strings.TrimSuffix(file, "/")))
	if err != nil {
		return true, err
	}
	// If the directory has a different device as parent, then it is a mountpoint.
	if stat.Sys().(*syscall.Stat_t).Dev != rootStat.Sys().(*syscall.Stat_t).Dev {
		return false, nil
	}

	return true, nil
}

```

**（3）NodeExpandVolume**

**简介**

负责node端的存储扩容操作。主要是在node上做相应操作，将存储的扩容信息同步到node上。



NodeExpandVolume resizes rbd volumes.



实际上，存储扩容分为两大步骤，第一步是csi的ControllerExpandVolume，主要负责将底层存储扩容；第二步是csi的NodeExpandVolume，当volumemode是filesystem时，主要负责将底层rbd image的扩容信息同步到rbd/nbd device，对xfs/ext文件系统进行扩展；当volumemode是block，则不用进行node端扩容操作。



**NodeExpandVolume**

主体流程：

（1）校验请求参数；

（2）判断指定路径是否为挂载点；

（3）获取devicePath；

（4）调用resizefs.NewResizeFs初始化resizer；

（5）调用resizer.Resize做进一步操作。

```
func (ns *NodeServer) NodeExpandVolume(ctx context.Context, req *csi.NodeExpandVolumeRequest) (*csi.NodeExpandVolumeResponse, error) {
	volumeID := req.GetVolumeId()
	if volumeID == "" {
		return nil, status.Error(codes.InvalidArgument, "volume ID must be provided")
	}
	volumePath := req.GetVolumePath()
	if volumePath == "" {
		return nil, status.Error(codes.InvalidArgument, "volume path must be provided")
	}

	if acquired := ns.VolumeLocks.TryAcquire(volumeID); !acquired {
		klog.Errorf(util.Log(ctx, util.VolumeOperationAlreadyExistsFmt), volumeID)
		return nil, status.Errorf(codes.Aborted, util.VolumeOperationAlreadyExistsFmt, volumeID)
	}
	defer ns.VolumeLocks.Release(volumeID)

	// volumePath is targetPath for block PVC and stagingPath for filesystem.
	// check the path is mountpoint or not, if it is
	// mountpoint treat this as block PVC or else it is filesystem PVC
	// TODO remove this once ceph-csi supports CSI v1.2.0 spec
	notMnt, err := mount.IsNotMountPoint(ns.mounter, volumePath)
	if err != nil {
		if os.IsNotExist(err) {
			return nil, status.Error(codes.NotFound, err.Error())
		}
		return nil, status.Error(codes.Internal, err.Error())
	}
	if !notMnt {
		return &csi.NodeExpandVolumeResponse{}, nil
	}

	devicePath, err := getDevicePath(ctx, volumePath)
	if err != nil {
		return nil, status.Error(codes.Internal, err.Error())
	}

	diskMounter := &mount.SafeFormatAndMount{Interface: ns.mounter, Exec: utilexec.New()}
	// TODO check size and return success or error
	volumePath += "/" + volumeID
	resizer := resizefs.NewResizeFs(diskMounter)
	ok, err := resizer.Resize(devicePath, volumePath)
	if !ok {
		return nil, fmt.Errorf("rbd: resize failed on path %s, error: %v", req.GetVolumePath(), err)
	}
	return &csi.NodeExpandVolumeResponse{}, nil
}

```

###### resizer.Resize

根据相应的文件系统格式，调用相应的resize方法。

```
func (resizefs *ResizeFs) Resize(devicePath string, deviceMountPath string) (bool, error) {
	format, err := resizefs.mounter.GetDiskFormat(devicePath)

	if err != nil {
		formatErr := fmt.Errorf("ResizeFS.Resize - error checking format for device %s: %v", devicePath, err)
		return false, formatErr
	}

	// If disk has no format, there is no need to resize the disk because mkfs.*
	// by default will use whole disk anyways.
	if format == "" {
		return false, nil
	}

	klog.V(3).Infof("ResizeFS.Resize - Expanding mounted volume %s", devicePath)
	switch format {
	case "ext3", "ext4":
		return resizefs.extResize(devicePath)
	case "xfs":
		return resizefs.xfsResize(deviceMountPath)
	}
	return false, fmt.Errorf("ResizeFS.Resize - resize of format %s is not supported for device %s mounted at %s", format, devicePath, deviceMountPath)
}

```

###### xfsResize

xfs文件系统使用xfs\_growfs命令。

```
func (resizefs *ResizeFs) xfsResize(deviceMountPath string) (bool, error) {
	args := []string{"-d", deviceMountPath}
	output, err := resizefs.mounter.Exec.Command("xfs_growfs", args...).CombinedOutput()

	if err == nil {
		klog.V(2).Infof("Device %s resized successfully", deviceMountPath)
		return true, nil
	}

	resizeError := fmt.Errorf("resize of device %s failed: %v. xfs_growfs output: %s", deviceMountPath, err, string(output))
	return false, resizeError
}

```

###### extResize

ext文件系统使用resize2fs命令。

```
func (resizefs *ResizeFs) extResize(devicePath string) (bool, error) {
	output, err := resizefs.mounter.Exec.Command("resize2fs", devicePath).CombinedOutput()
	if err == nil {
		klog.V(2).Infof("Device %s resized successfully", devicePath)
		return true, nil
	}

	resizeError := fmt.Errorf("resize of device %s failed: %v. resize2fs output: %s", devicePath, err, string(output))
	return false, resizeError

}

```

**rbd driver-nodeserver分析（上）-小结**

这节分析了NodeGetCapabilities、NodeGetVolumeStats、NodeExpandVolume方法，作用分别如下：



NodeGetCapabilities：获取ceph-csi driver的能力。

NodeGetVolumeStats：探测挂载存储的状态，并返回该存储的相关metrics给kubelet。

NodeExpandVolume：在node上做相应操作，将存储的扩容信息同步到node上。



