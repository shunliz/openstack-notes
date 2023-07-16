# **ceph-csi源码分析（6）-rbd driver-nodeserver分析（下）**

当ceph-csi组件启动时指定的driver type为rbd时，会启动rbd driver相关的服务。然后再根据controllerserver、nodeserver的参数配置，决定启动ControllerServer与IdentityServer，或NodeServer与IdentityServer。

基于tag v3.0.0

[https://github.com/ceph/ceph-csi/releases/tag/v3.0.0](https://github.com/ceph/ceph-csi/releases/tag/v3.0.0)

rbd driver分析将分为4个部分，分别是服务入口分析、controllerserver分析、nodeserver分析与IdentityServer分析。  
![](/assets/compute-container-k8s-cephcsi13633361.png)nodeserver主要包括了NodeGetCapabilities（获取driver能力）、NodeGetVolumeStats（存储探测及metrics获取）、NodeStageVolume（map rbd与mount stagingPath）、NodePublishVolume（mount targetPath）、NodeUnpublishVolume（umount targetPath）、NodeUnstageVolume（umount stagingPath与unmap rbd）、NodeExpandVolume（node端存储扩容）操作，将一一进行分析。这节进行NodeStageVolume、NodePublishVolume、NodeUnpublishVolume、NodeUnstageVolume的分析。



**nodeserver分析（下）**

ceph rbd挂载知识讲解

rbd image map成块设备，主要有两种方式：\(1\)通过RBD Kernel Module，\(2\)通过RBD-NBD。参考：https://www.jianshu.com/p/bb9d14bd897c 、http://xiaqunfeng.cc/2017/06/07/Map-RBD-Devices-on-NBD/



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








