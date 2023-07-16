# **ceph-csi源码分析（6）-rbd driver-nodeserver分析（下）**

当ceph-csi组件启动时指定的driver type为rbd时，会启动rbd driver相关的服务。然后再根据controllerserver、nodeserver的参数配置，决定启动ControllerServer与IdentityServer，或NodeServer与IdentityServer。

基于tag v3.0.0

[https://github.com/ceph/ceph-csi/releases/tag/v3.0.0](https://github.com/ceph/ceph-csi/releases/tag/v3.0.0)

rbd driver分析将分为4个部分，分别是服务入口分析、controllerserver分析、nodeserver分析与IdentityServer分析。  
![](/assets/compute-container-k8s-cephcsi13633361.png)nodeserver主要包括了NodeGetCapabilities（获取driver能力）、NodeGetVolumeStats（存储探测及metrics获取）、NodeStageVolume（map rbd与mount stagingPath）、NodePublishVolume（mount targetPath）、NodeUnpublishVolume（umount targetPath）、NodeUnstageVolume（umount stagingPath与unmap rbd）、NodeExpandVolume（node端存储扩容）操作，将一一进行分析。这节进行NodeStageVolume、NodePublishVolume、NodeUnpublishVolume、NodeUnstageVolume的分析。

**nodeserver分析（下）**

ceph rbd挂载知识讲解

rbd image map成块设备，主要有两种方式：\(1\)通过RBD Kernel Module，\(2\)通过RBD-NBD。参考：[https://www.jianshu.com/p/bb9d14bd897c](https://www.jianshu.com/p/bb9d14bd897c) 、[http://xiaqunfeng.cc/2017/06/07/Map-RBD-Devices-on-NBD/](http://xiaqunfeng.cc/2017/06/07/Map-RBD-Devices-on-NBD/)

一个ceph rbd image挂载给pod，一共分为2个步骤，分别如下：

1.kubelet组件调用rbdType-nodeserver-ceph-csi的NodeStageVolume方法，将rbd image map到node上的rbd/nbd device，然后将rbd device格式化并mount到staging path；

2.kubelet组件调用rbdType-nodeserver-ceph-csi的NodePublishVolume方法，将上一步骤中的staging path mount到target path。

**ceph rbd解除挂载知识讲解**

一个ceph rbd image从pod中解除挂载，一共分为2个步骤，分别如下：

1.kubelet组件调用rbdType-nodeserver-ceph-csi的NodeUnpublishVolume方法，解除掉stagingPath与targetPath的挂载关系。

2.kubelet组件调用rbdType-nodeserver-ceph-csi的NodeUnstageVolume方法，先解除掉targetPath与rbd/nbd device的挂载关系，然后再unmap掉rbd/nbd device（即解除掉node端rbd/nbd device与ceph rbd image的挂载）。

rbd image挂载给pod后，node上会出现2个mount关系，示例如下：

\# mount \| grep nbd

/dev/nbd0 on /home/cld/kubernetes/lib/kubelet/plugins/kubernetes.io/csi/pv/pvc-e2104b0f-774e-420e-a388-1705344084a4/globalmount/0001-0024-0bba3be9-0a1c-41db-a619-26ffea20161e-0000000000000004-40b130e1-a630-11eb-8bea-246e968ec20c type xfs \(rw,relatime,nouuid,attr2,inode64,noquota,\_netdev\)

/dev/nbd0 on /home/cld/kubernetes/lib/kubelet/pods/80114f88-2b09-440c-aec2-54c16efe6923/volumes/kubernetes.io~csi/pvc-e2104b0f-774e-420e-a388-1705344084a4/mount type xfs \(rw,relatime,nouuid,attr2,inode64,noquota,\_netdev\)

其中/home/cld/kubernetes/lib/kubelet/plugins/kubernetes.io/csi/pv/pvc-e2104b0f-774e-420e-a388-1705344084a4/globalmount/0001-0024-0bba3be9-0a1c-41db-a619-26ffea20161e-0000000000000004-40b130e1-a630-11eb-8bea-246e968ec20c为staging path；而/home/cld/kubernetes/lib/kubelet/pods/80114f88-2b09-440c-aec2-54c16efe6923/volumes/kubernetes.io~csi/pvc-e2104b0f-774e-420e-a388-1705344084a4/mount为target path，/dev/nbd0为nbd device。

注意

一个rbd image挂载给一个node上的多个pod时，NodeStageVolume方法只会被调用一次，NodePublishVolume会被调用多次，即出现该情况时，staging path只有一个，而target path会有多个。你可以这样理解，staging path对应的是rbd image，而target path对应的是pod，所以当一个rbd image挂载给一个node上的多个pod时，staging path只有一个，而target path会有多个。

解除挂载也同理，当挂载了某个rbd image的所有pod都被删除，NodeUnstageVolume方法才会被调用。

**（4）NodeStageVolume**

简介

将rbd image map到node上的rbd/nbd device，并格式化后挂载到staging path。

NodeStageVolume mounts the volume to a staging path on the node.

Stash image metadata under staging path

Map the image \(creates a device\)

Create the staging file/directory under staging path

Stage the device \(mount the device mapped for image\)

主要步骤：

（1）将rbd image map到node上的rbd/nbd device；

（2）将rbd device格式化（volumeMode为block时，不用格式化），并mount到staging path。

NodeStageVolume

NodeStageVolume主体流程：

（1）校验请求参数、校验AccessMode；

（2）从请求参数中获取volID；

（3）根据secret构建ceph请求凭证（secret由kubelet传入）；

（4）检查stagingPath是否存在，是否已经mount；

（5）根据volID从volume journal中获取image name；

（6）在stagingParentPath下创建image-meta.json，用于存储image的元数据；

（7）调用ns.stageTransaction做map与mount操作。

    //ceph-csi/internal/rbd/nodeserver.go

    func (ns *NodeServer) NodeStageVolume(ctx context.Context, req *csi.NodeStageVolumeRequest) (*csi.NodeStageVolumeResponse, error) {
        // （1）校验请求参数
        if err := util.ValidateNodeStageVolumeRequest(req); err != nil {
            return nil, err
        }

        // 校验AccessMode
        isBlock := req.GetVolumeCapability().GetBlock() != nil
        disableInUseChecks := false
        // MULTI_NODE_MULTI_WRITER is supported by default for Block access type volumes
        if req.VolumeCapability.AccessMode.Mode == csi.VolumeCapability_AccessMode_MULTI_NODE_MULTI_WRITER {
            if !isBlock {
                klog.Warningf(util.Log(ctx, "MULTI_NODE_MULTI_WRITER currently only supported with volumes of access type `block`, invalid AccessMode for volume: %v"), req.GetVolumeId())
                return nil, status.Error(codes.InvalidArgument, "rbd: RWX access mode request is only valid for volumes with access type `block`")
            }

            disableInUseChecks = true
        }

        // （2）从请求参数中获取volID
        volID := req.GetVolumeId()

        // （3）根据secret构建ceph请求凭证
        cr, err := util.NewUserCredentials(req.GetSecrets())
        if err != nil {
            return nil, status.Error(codes.Internal, err.Error())
        }
        defer cr.DeleteCredentials()

        if acquired := ns.VolumeLocks.TryAcquire(volID); !acquired {
            klog.Errorf(util.Log(ctx, util.VolumeOperationAlreadyExistsFmt), volID)
            return nil, status.Errorf(codes.Aborted, util.VolumeOperationAlreadyExistsFmt, volID)
        }
        defer ns.VolumeLocks.Release(volID)

        stagingParentPath := req.GetStagingTargetPath()
        stagingTargetPath := stagingParentPath + "/" + volID

        // check is it a static volume
        staticVol := false
        val, ok := req.GetVolumeContext()["staticVolume"]
        if ok {
            if staticVol, err = strconv.ParseBool(val); err != nil {
                return nil, status.Error(codes.InvalidArgument, err.Error())
            }
        }

        // （4）检查stagingPath是否存在，是否已经mount
        var isNotMnt bool
        // check if stagingPath is already mounted
        isNotMnt, err = mount.IsNotMountPoint(ns.mounter, stagingTargetPath)
        if err != nil && !os.IsNotExist(err) {
            return nil, status.Error(codes.Internal, err.Error())
        }

        if !isNotMnt {
            util.DebugLog(ctx, "rbd: volume %s is already mounted to %s, skipping", volID, stagingTargetPath)
            return &csi.NodeStageVolumeResponse{}, nil
        }

        volOptions, err := genVolFromVolumeOptions(ctx, req.GetVolumeContext(), req.GetSecrets(), disableInUseChecks)
        if err != nil {
            return nil, status.Error(codes.Internal, err.Error())
        }

        // （5）根据volID从volume journal中获取image name
        // get rbd image name from the volume journal
        // for static volumes, the image name is actually the volume ID itself
        switch {
        case staticVol:
            volOptions.RbdImageName = volID
        default:
            var vi util.CSIIdentifier
            var imageAttributes *journal.ImageAttributes
            err = vi.DecomposeCSIID(volID)
            if err != nil {
                err = fmt.Errorf("error decoding volume ID (%s) (%s)", err, volID)
                return nil, status.Error(codes.Internal, err.Error())
            }

            j, err2 := volJournal.Connect(volOptions.Monitors, cr)
            if err2 != nil {
                klog.Errorf(
                    util.Log(ctx, "failed to establish cluster connection: %v"),
                    err2)
                return nil, status.Error(codes.Internal, err.Error())
            }
            defer j.Destroy()

            imageAttributes, err = j.GetImageAttributes(
                ctx, volOptions.Pool, vi.ObjectUUID, false)
            if err != nil {
                err = fmt.Errorf("error fetching image attributes for volume ID (%s) (%s)", err, volID)
                return nil, status.Error(codes.Internal, err.Error())
            }
            volOptions.RbdImageName = imageAttributes.ImageName
        }

        volOptions.VolID = volID
        transaction := stageTransaction{}

        // （6）在stagingParentPath下创建image-meta.json，用于存储image的元数据
        // Stash image details prior to mapping the image (useful during Unstage as it has no
        // voloptions passed to the RPC as per the CSI spec)
        err = stashRBDImageMetadata(volOptions, stagingParentPath)
        if err != nil {
            return nil, status.Error(codes.Internal, err.Error())
        }
        defer func() {
            if err != nil {
                ns.undoStagingTransaction(ctx, req, transaction)
            }
        }()

        // （7）调用ns.stageTransaction做map/mount操作
        // perform the actual staging and if this fails, have undoStagingTransaction
        // cleans up for us
        transaction, err = ns.stageTransaction(ctx, req, volOptions, staticVol)
        if err != nil {
            return nil, status.Error(codes.Internal, err.Error())
        }

        util.DebugLog(ctx, "rbd: successfully mounted volume %s to stagingTargetPath %s", req.GetVolumeId(), stagingTargetPath)

        return &csi.NodeStageVolumeResponse{}, nil
    }

**1.ValidateNodeStageVolumeRequest**

ValidateNodeStageVolumeRequest校验了如下内容：

（1）volume capability参数不能为空；

（2）volume ID参数不能为空；

（3）staging target path（临时目录）参数不能为空；

（4）stage secrets参数不能为空；

（5）staging path（临时目录）是否存在于dnode上。

##### 2.stashRBDImageMetadata

stashRBDImageMetadata在stagingParentPath下创建image-meta.json，用于存储image的元数据。

##### 3.ns.stageTransaction

主要流程：  
（1）调用attachRBDImage将rbd device map到dnode；  
（2）调用ns.mountVolumeToStagePath将dnode上的rbd device格式化后 mount到StagePath。

```
//ceph-csi/internal/rbd/nodeserver.go

func (ns *NodeServer) stageTransaction(ctx context.Context, req *csi.NodeStageVolumeRequest, volOptions *rbdVolume, staticVol bool) (stageTransaction, error) {
	transaction := stageTransaction{}

	var err error
	var readOnly bool
	var feature bool

	var cr *util.Credentials
	cr, err = util.NewUserCredentials(req.GetSecrets())
	if err != nil {
		return transaction, err
	}
	defer cr.DeleteCredentials()

	err = volOptions.Connect(cr)
	if err != nil {
		klog.Errorf(util.Log(ctx, "failed to connect to volume %v: %v"), volOptions.RbdImageName, err)
		return transaction, err
	}
	defer volOptions.Destroy()

	// Allow image to be mounted on multiple nodes if it is ROX
	if req.VolumeCapability.AccessMode.Mode == csi.VolumeCapability_AccessMode_MULTI_NODE_READER_ONLY {
		util.ExtendedLog(ctx, "setting disableInUseChecks on rbd volume to: %v", req.GetVolumeId)
		volOptions.DisableInUseChecks = true
		volOptions.readOnly = true
	}

	if kernelRelease == "" {
		// fetch the current running kernel info
		kernelRelease, err = util.GetKernelVersion()
		if err != nil {
			return transaction, err
		}
	}
	if !util.CheckKernelSupport(kernelRelease, deepFlattenSupport) {
		if !skipForceFlatten {
			feature, err = volOptions.checkImageChainHasFeature(ctx, librbd.FeatureDeepFlatten)
			if err != nil {
				return transaction, err
			}
			if feature {
				err = volOptions.flattenRbdImage(ctx, cr, true, rbdHardMaxCloneDepth, rbdSoftMaxCloneDepth)
				if err != nil {
					return transaction, err
				}
			}
		}
	}
	// Mapping RBD image
	var devicePath string
	devicePath, err = attachRBDImage(ctx, volOptions, cr)
	if err != nil {
		return transaction, err
	}
	transaction.devicePath = devicePath
	util.DebugLog(ctx, "rbd image: %s/%s was successfully mapped at %s\n",
		req.GetVolumeId(), volOptions.Pool, devicePath)

	if volOptions.Encrypted {
		devicePath, err = ns.processEncryptedDevice(ctx, volOptions, devicePath)
		if err != nil {
			return transaction, err
		}
		transaction.isEncrypted = true
	}

	stagingTargetPath := getStagingTargetPath(req)

	isBlock := req.GetVolumeCapability().GetBlock() != nil
	err = ns.createStageMountPoint(ctx, stagingTargetPath, isBlock)
	if err != nil {
		return transaction, err
	}

	transaction.isStagePathCreated = true

	// nodeStage Path
	readOnly, err = ns.mountVolumeToStagePath(ctx, req, staticVol, stagingTargetPath, devicePath)
	if err != nil {
		return transaction, err
	}
	transaction.isMounted = true

	if !readOnly {
		// #nosec - allow anyone to write inside the target path
		err = os.Chmod(stagingTargetPath, 0777)
	}
	return transaction, err
}

```

3.1 attachRBDImage

attachRBDImage主要流程：

（1）调用waitForPath判断image是否已经map到该node上；

（2）没有map到该node上时，调用waitForrbdImage判断image是否存在，是否已被使用；

（3）调用createPath将image map到node上。

```
//ceph-csi/internal/rbd/rbd_attach.go

func attachRBDImage(ctx context.Context, volOptions *rbdVolume, cr *util.Credentials) (string, error) {
	var err error

	image := volOptions.RbdImageName
	useNBD := false
	if volOptions.Mounter == rbdTonbd && hasNBD {
		useNBD = true
	}

	devicePath, found := waitForPath(ctx, volOptions.Pool, image, 1, useNBD)
	if !found {
		backoff := wait.Backoff{
			Duration: rbdImageWatcherInitDelay,
			Factor:   rbdImageWatcherFactor,
			Steps:    rbdImageWatcherSteps,
		}

		err = waitForrbdImage(ctx, backoff, volOptions)

		if err != nil {
			return "", err
		}
		devicePath, err = createPath(ctx, volOptions, cr)
	}

	return devicePath, err
}

```

createPath拼接ceph命令，然后执行map命令，将rbd image map到dnode上成为rbd device。

rbd-nbd挂载模式，通过–device-type=nbd指定。

```
func createPath(ctx context.Context, volOpt *rbdVolume, cr *util.Credentials) (string, error) {
	isNbd := false
	imagePath := volOpt.String()

	util.TraceLog(ctx, "rbd: map mon %s", volOpt.Monitors)

	// Map options
	mapOptions := []string{
		"--id", cr.ID,
		"-m", volOpt.Monitors,
		"--keyfile=" + cr.KeyFile,
		"map", imagePath,
	}

	// Choose access protocol
	accessType := accessTypeKRbd
	if volOpt.Mounter == rbdTonbd && hasNBD {
		isNbd = true
		accessType = accessTypeNbd
	}

	// Update options with device type selection
	mapOptions = append(mapOptions, "--device-type", accessType)

	if volOpt.readOnly {
		mapOptions = append(mapOptions, "--read-only")
	}
	// Execute map
	stdout, stderr, err := util.ExecCommand(ctx, rbd, mapOptions...)
	if err != nil {
		klog.Warningf(util.Log(ctx, "rbd: map error %v, rbd output: %s"), err, stderr)
		// unmap rbd image if connection timeout
		if strings.Contains(err.Error(), rbdMapConnectionTimeout) {
			detErr := detachRBDImageOrDeviceSpec(ctx, imagePath, true, isNbd, volOpt.Encrypted, volOpt.VolID)
			if detErr != nil {
				klog.Warningf(util.Log(ctx, "rbd: %s unmap error %v"), imagePath, detErr)
			}
		}
		return "", fmt.Errorf("rbd: map failed with error %v, rbd error output: %s", err, stderr)
	}
	devicePath := strings.TrimSuffix(stdout, "\n")

	return devicePath, nil
}

```

###### 3.2 mountVolumeToStagePath

主体流程：  
（1）当volumeMode为Filesystem时，运行mkfs格式化rbd device；  
（2）将rbd device挂载到stagingPath。





**（5）NodePublishVolume**

**简介**

将NodeStageVolume方法中的staging path，mount到target path。



NodeStageVolume将rbd image map到dnode上成为device后，随即将device mount到了一个staging path。



NodePublishVolume将stagingPath mount到target path。



stagingPath示例：

/home/cld/kubernetes/lib/kubelet/plugins/kubernetes.io/csi/pv/pvc-14ee5002-9d60-4ba3-a1d2-cc3800ee0893/globalmount/0001-0024-0bba3be9-0a1c-41db-a619-26ffea20161e-0000000000000004-1699e662-e83f-11ea-8e79-246e96907f74  



targetPath示例：  

/home/cld/kubernetes/lib/kubelet/pods/c14de522-0679-44b6-af8b-e1ba08b5b004/volumes/kubernetes.io~csi/pvc-14ee5002-9d60-4ba3-a1d2-cc3800ee0893/mount



**NodePublishVolume**

主体流程：

（1）校验请求参数；

（2）检查target path是否存在，不存在则创建；

（3）将staging path挂载到target path。

**1.ValidateNodePublishVolumeRequest**

ValidateNodePublishVolumeRequest主要是校验部分请求参数，校验volume capability/volume ID/target path/staging target path不能为空。

###### 2.createTargetMountPath

createTargetMountPath主要是检查mount path是否存在，不存在则创建

###### 3.mountVolume

mountVolume主要是拼凑mount命令，将staging path挂载到target path





**（6）NodeUnpublishVolume**

**简介**

解除掉stagingPath到targetPath的挂载。



NodeUnpublishVolume unmounts the volume from the target path.



**NodeUnpublishVolume**

主体流程：

（1）校验请求参数；

（2）判断指定路径是否为挂载点；

（3）解除掉stagingPath到targetPath的挂载；

（4）删除targetPath目录及其包含的任何子目录。







**（7）NodeUnstageVolume**

**简介**

先解除掉targetPath到rbd/nbd device的挂载，然后再unmap掉rbd/nbd device（即解除掉node端rbd/nbd device与ceph rbd image的挂载）。



NodeUnstageVolume unstages the volume from the staging path.



**NodeUnstageVolume**

主体流程：

（1）校验请求参数；

（2）判断stagingTargetPath是否存在；

（3）将stagingTargetPath unmount rbd device；

（4）删除stagingTargetPath；

（5）从stagingParentPath的image-meta.json文件中读取image的元数据；

（6）unmap rbd device；

（7）删除该image对应的元数据，即image-meta.json文件。



###### 1.lookupRBDImageMetadataStash

从stagingParentPath的image-meta.json文件中读取image的元数据。

###### 2.detachRBDImageOrDeviceSpec

拼凑unmap命令，进行unmap rbd/nbd device。



```
//ceph-csi/internal/rbd/rbd_attach.go

func detachRBDImageOrDeviceSpec(ctx context.Context, imageOrDeviceSpec string, isImageSpec, ndbType, encrypted bool, volumeID string) error {
	if encrypted {
		mapperFile, mapperPath := util.VolumeMapper(volumeID)
		mappedDevice, mapper, err := util.DeviceEncryptionStatus(ctx, mapperPath)
		if err != nil {
			klog.Errorf(util.Log(ctx, "error determining LUKS device on %s, %s: %s"),
				mapperPath, imageOrDeviceSpec, err)
			return err
		}
		if len(mapper) > 0 {
			// mapper found, so it is open Luks device
			err = util.CloseEncryptedVolume(ctx, mapperFile)
			if err != nil {
				klog.Errorf(util.Log(ctx, "error closing LUKS device on %s, %s: %s"),
					mapperPath, imageOrDeviceSpec, err)
				return err
			}
			imageOrDeviceSpec = mappedDevice
		}
	}

	accessType := accessTypeKRbd
	if ndbType {
		accessType = accessTypeNbd
	}
	options := []string{"unmap", "--device-type", accessType, imageOrDeviceSpec}

	_, stderr, err := util.ExecCommand(ctx, rbd, options...)
	if err != nil {
		// Messages for krbd and nbd differ, hence checking either of them for missing mapping
		// This is not applicable when a device path is passed in
		if isImageSpec &&
			(strings.Contains(stderr, fmt.Sprintf(rbdUnmapCmdkRbdMissingMap, imageOrDeviceSpec)) ||
				strings.Contains(stderr, fmt.Sprintf(rbdUnmapCmdNbdMissingMap, imageOrDeviceSpec))) {
			// Devices found not to be mapped are treated as a successful detach
			util.TraceLog(ctx, "image or device spec (%s) not mapped", imageOrDeviceSpec)
			return nil
		}
		return fmt.Errorf("rbd: unmap for spec (%s) failed (%v): (%s)", imageOrDeviceSpec, err, stderr)
	}

	return nil
}

```

###### 3.cleanupRBDImageMetadataStash

删除该image对应的元数据，即image-meta.json文件。





至此，rbd driver-nodeserver的分析已经全部完成，下面做个总结。



**rbd driver-nodeserver分析总结**

（1）nodeserver主要包括了NodeGetCapabilities、NodeGetVolumeStats、NodeStageVolume、NodePublishVolume、NodeUnpublishVolume、NodeUnstageVolume、NodeExpandVolume方法，作用分别如下：



NodeGetCapabilities：获取ceph-csi driver的能力。

NodeGetVolumeStats：探测挂载存储的状态，并返回该存储的相关metrics给kubelet。

NodeExpandVolume：在node上做相应操作，将存储的扩容信息同步到node上。

NodeStageVolume：将rbd image map到node上的rbd/nbd device，并格式化后挂载到staging path。

NodePublishVolume：将NodeStageVolume方法中的staging path，mount到target path。

NodeUnpublishVolume：解除掉stagingPath到targetPath的挂载。

NodeUnstageVolume：先解除掉targetPath到rbd/nbd device的挂载，然后再unmap掉rbd/nbd device（即解除掉node端rbd/nbd device与ceph rbd image的挂载）。

（2）在kubelet调用NodeExpandVolume、NodeStageVolume、NodeUnstageVolume等方法前，会先调用NodeGetCapabilities来获取该ceph-csi driver的能力，看是否支持对这些方法的调用。



（3）kubelet定时循环调用NodeGetVolumeStats，获取volume相关指标。



（4）存储扩容分为两大步骤，第一步是csi的ControllerExpandVolume，主要负责将底层存储扩容；第二步是csi的NodeExpandVolume，当volumemode是filesystem时，主要负责将底层rbd image的扩容信息同步到rbd/nbd device，对xfs/ext文件系统进行扩展；当volumemode是block，则不用进行node端扩容操作。



（5）一个rbd image挂载给一个node上的多个pod时，NodeStageVolume方法只会被调用一次，NodePublishVolume会被调用多次，即出现该情况时，staging path只有一个，而target path会有多个。你可以这样理解，staging path对应的是rbd image，而target path对应的是pod，所以当一个rbd image挂载给一个node上的多个pod时，staging path只有一个，而target path会有多个。解除挂载也同理，当挂载了某个rbd image的所有pod都被删除，NodeUnstageVolume方法才会被调用。

6560674

