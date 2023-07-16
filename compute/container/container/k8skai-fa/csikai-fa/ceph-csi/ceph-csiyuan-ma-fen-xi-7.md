## ceph-csi源码分析（4）-rbd driver-controllerserver分析

当ceph-csi组件启动时指定的driver type为rbd时，会启动rbd driver相关的服务。然后再根据controllerserver、nodeserver的参数配置，决定启动ControllerServer与IdentityServer，或NodeServer与IdentityServer。

基于tag v3.0.0

[https://github.com/ceph/ceph-csi/releases/tag/v3.0.0](https://github.com/ceph/ceph-csi/releases/tag/v3.0.0)

rbd driver分析将分为4个部分，分别是服务入口分析、controllerserver分析、nodeserver分析与IdentityServer分析。

![](/assets/compute-container-k8s-cephcsi13633361.png)这节进行controllerserver分析，controllerserver主要包括了CreateVolume（创建存储）、DeleteVolume（删除存储）、ControllerExpandVolume（存储扩容）、CreateSnapshot（创建存储快照）、DeleteSnapshot（删除存储快照）操作，这里主要对CreateVolume（创建存储）、DeleteVolume（删除存储）、ControllerExpandVolume（存储扩容）进行分析。



**controllerserver分析**

**（1）CreateVolume**

**简介**

调用ceph存储后端，创建存储（rbd image）。



CreateVolume creates the volume in backend.



大致步骤：

（1）检查该driver是否支持存储的创建，不支持则直接返回错误；

（2）构建ceph请求凭证，生成volume ID；

（3）调用ceph存储后端，创建存储。



三种创建来源：

（1）从已有image拷贝；

（2）根据快照创建image；

（3）直接创建新的image。



**CreateVolume**

主体流程：

（1）检查该driver是否支持存储的创建，不支持则直接返回错误；

（2）根据secret构建ceph请求凭证（secret由external-provisioner组件传入）；

（3）处理请求参数，并转换为rbdVol结构体（根据请求参数clusterID，从本地ceph配置文件中读取该clusterID对应的monitor信息，将clusterID与monitor赋值给rbdVol结构体；检查请求参数imageFeatures，只支持Layering；根据请求参数VolumeCapabilities决定在存储挂载时是否启动in-use检查，即多节点挂载检查；存储大小只支持整数，对小数做进一处理）；

（4）检查并获取锁（同名存储在同一时间，只能做创建、删除、扩容操作中的一个，通过该锁来保证互斥）；

（5）创建与Ceph集群的连接；

（6）生成RbdImageName与ReservedID，并生成volume ID；

（7）调用createBackingImage去创建image；

（8）释放锁。

（暂时跳过从已有image拷贝与根据快照创建image的分析）

```
//ceph-csi/internal/rbd/controllerserver.go

func (cs *ControllerServer) CreateVolume(ctx context.Context, req *csi.CreateVolumeRequest) (*csi.CreateVolumeResponse, error) {
    // （1）校验请求参数；  
	if err := cs.validateVolumeReq(ctx, req); err != nil {
		return nil, err
	}

	// （2）根据secret构建ceph请求凭证； 
	cr, err := util.NewUserCredentials(req.GetSecrets())
	if err != nil {
		return nil, status.Error(codes.Internal, err.Error())
	}
	defer cr.DeleteCredentials()
    
    // （3）将请求参数转换为rbdVol结构体； 
	rbdVol, err := cs.parseVolCreateRequest(ctx, req)
	if err != nil {
		return nil, err
	}
	defer rbdVol.Destroy()
	// Existence and conflict checks
	if acquired := cs.VolumeLocks.TryAcquire(req.GetName()); !acquired {
		klog.Errorf(util.Log(ctx, util.VolumeOperationAlreadyExistsFmt), req.GetName())
		return nil, status.Errorf(codes.Aborted, util.VolumeOperationAlreadyExistsFmt, req.GetName())
	}
	defer cs.VolumeLocks.Release(req.GetName())
    
    // （4）创建与Ceph集群的连接； 
	err = rbdVol.Connect(cr)
	if err != nil {
		klog.Errorf(util.Log(ctx, "failed to connect to volume %v: %v"), rbdVol.RbdImageName, err)
		return nil, status.Error(codes.Internal, err.Error())
	}

	parentVol, rbdSnap, err := checkContentSource(ctx, req, cr)
	if err != nil {
		return nil, err
	}

	found, err := rbdVol.Exists(ctx, parentVol)
	if err != nil {
		return nil, getGRPCErrorForCreateVolume(err)
	}
	if found {
		if rbdSnap != nil {
			// check if image depth is reached limit and requires flatten
			err = checkFlatten(ctx, rbdVol, cr)
			if err != nil {
				return nil, err
			}
		}
		return buildCreateVolumeResponse(ctx, req, rbdVol)
	}

	err = validateRequestedVolumeSize(rbdVol, parentVol, rbdSnap, cr)
	if err != nil {
		return nil, err
	}

	err = flattenParentImage(ctx, parentVol, cr)
	if err != nil {
		return nil, err
	}
    
    // （5）生成rbd image name并生成volume ID； 
	err = reserveVol(ctx, rbdVol, rbdSnap, cr)
	if err != nil {
		return nil, status.Error(codes.Internal, err.Error())
	}
	defer func() {
		if err != nil {
			if !errors.Is(err, ErrFlattenInProgress) {
				errDefer := undoVolReservation(ctx, rbdVol, cr)
				if errDefer != nil {
					klog.Warningf(util.Log(ctx, "failed undoing reservation of volume: %s (%s)"), req.GetName(), errDefer)
				}
			}
		}
	}()
    
    // （6）调用createBackingImage去创建image。 
	err = cs.createBackingImage(ctx, cr, rbdVol, parentVol, rbdSnap)
	if err != nil {
		if errors.Is(err, ErrFlattenInProgress) {
			return nil, status.Error(codes.Aborted, err.Error())
		}
		return nil, err
	}

	volumeContext := req.GetParameters()
	volumeContext["pool"] = rbdVol.Pool
	volumeContext["journalPool"] = rbdVol.JournalPool
	volumeContext["imageName"] = rbdVol.RbdImageName
	volume := &csi.Volume{
		VolumeId:      rbdVol.VolID,
		CapacityBytes: rbdVol.VolSize,
		VolumeContext: volumeContext,
		ContentSource: req.GetVolumeContentSource(),
	}
	if rbdVol.Topology != nil {
		volume.AccessibleTopology =
			[]*csi.Topology{
				{
					Segments: rbdVol.Topology,
				},
			}
	}
	return &csi.CreateVolumeResponse{Volume: volume}, nil
}

```



###### 1 reserveVol

用于生成RbdImageName与ReservedID，并生成volume ID。调用j.ReserveName获取imageName，调用util.GenerateVolID生成volume ID。

```
//ceph-csi/internal/rbd/rbd_journal.go

// reserveVol is a helper routine to request a rbdVolume name reservation and generate the
// volume ID for the generated name.
func reserveVol(ctx context.Context, rbdVol *rbdVolume, rbdSnap *rbdSnapshot, cr *util.Credentials) error {
	var (
		err error
	)

	err = updateTopologyConstraints(rbdVol, rbdSnap)
	if err != nil {
		return err
	}

	journalPoolID, imagePoolID, err := util.GetPoolIDs(ctx, rbdVol.Monitors, rbdVol.JournalPool, rbdVol.Pool, cr)
	if err != nil {
		return err
	}

	kmsID := ""
	if rbdVol.Encrypted {
		kmsID = rbdVol.KMS.GetID()
	}

	j, err := volJournal.Connect(rbdVol.Monitors, cr)
	if err != nil {
		return err
	}
	defer j.Destroy()
    
    // 生成rbd image name
	rbdVol.ReservedID, rbdVol.RbdImageName, err = j.ReserveName(
		ctx, rbdVol.JournalPool, journalPoolID, rbdVol.Pool, imagePoolID,
		rbdVol.RequestName, rbdVol.NamePrefix, "", kmsID)
	if err != nil {
		return err
	}
    
    // 生成volume ID
	rbdVol.VolID, err = util.GenerateVolID(ctx, rbdVol.Monitors, cr, imagePoolID, rbdVol.Pool,
		rbdVol.ClusterID, rbdVol.ReservedID, volIDVersion)
	if err != nil {
		return err
	}

	util.DebugLog(ctx, "generated Volume ID (%s) and image name (%s) for request name (%s)",
		rbdVol.VolID, rbdVol.RbdImageName, rbdVol.RequestName)

	return nil
}

```

###### 1.1 GenerateVolID

调用vi.ComposeCSIID\(\)来生成volume ID。

```
//ceph-csi/internal/util/util.go

func GenerateVolID(ctx context.Context, monitors string, cr *Credentials, locationID int64, pool, clusterID, objUUID string, volIDVersion uint16) (string, error) {
	var err error

	if locationID == InvalidPoolID {
		locationID, err = GetPoolID(monitors, cr, pool)
		if err != nil {
			return "", err
		}
	}

	// generate the volume ID to return to the CO system
	vi := CSIIdentifier{
		LocationID:      locationID,
		EncodingVersion: volIDVersion,
		ClusterID:       clusterID,
		ObjectUUID:      objUUID,
	}

	volID, err := vi.ComposeCSIID()

	return volID, err
}

```

vi.ComposeCSIID\(\)为生成volume ID的方法，volume ID由`csi_id_version`+`length of clusterID`+`clusterID`+`poolID`+`ObjectUUID`组成，共64bytes。

```
//ceph-csi/internal/util/volid.go

/*
ComposeCSIID composes a CSI ID from passed in parameters.
Version 1 of the encoding scheme is as follows,
	[csi_id_version=1:4byte] + [-:1byte]
	[length of clusterID=1:4byte] + [-:1byte]
	[clusterID:36bytes (MAX)] + [-:1byte]
	[poolID:16bytes] + [-:1byte]
	[ObjectUUID:36bytes]

	Total of constant field lengths, including '-' field separators would hence be,
	4+1+4+1+1+16+1+36 = 64
*/
func (ci CSIIdentifier) ComposeCSIID() (string, error) {
	buf16 := make([]byte, 2)
	buf64 := make([]byte, 8)

	if (knownFieldSize + len(ci.ClusterID)) > maxVolIDLen {
		return "", errors.New("CSI ID encoding length overflow")
	}

	if len(ci.ObjectUUID) != uuidSize {
		return "", errors.New("CSI ID invalid object uuid")
	}

	binary.BigEndian.PutUint16(buf16, ci.EncodingVersion)
	versionEncodedHex := hex.EncodeToString(buf16)

	binary.BigEndian.PutUint16(buf16, uint16(len(ci.ClusterID)))
	clusterIDLength := hex.EncodeToString(buf16)

	binary.BigEndian.PutUint64(buf64, uint64(ci.LocationID))
	poolIDEncodedHex := hex.EncodeToString(buf64)

	return strings.Join([]string{versionEncodedHex, clusterIDLength, ci.ClusterID,
		poolIDEncodedHex, ci.ObjectUUID}, "-"), nil
}

```

###### 2 createBackingImage

主要流程：  
（1）调用createImage创建image；  
（2）调用j.StoreImageID存储image ID等信息进omap。

```
//ceph-csi/internal/rbd/controllerserver.go

func (cs *ControllerServer) createBackingImage(ctx context.Context, cr *util.Credentials, rbdVol, parentVol *rbdVolume, rbdSnap *rbdSnapshot) error {
	var err error

	var j = &journal.Connection{}
	j, err = volJournal.Connect(rbdVol.Monitors, cr)
	if err != nil {
		return status.Error(codes.Internal, err.Error())
	}
	defer j.Destroy()

	// nolint:gocritic // this ifElseChain can not be rewritten to a switch statement
	if rbdSnap != nil {
		if err = cs.OperationLocks.GetRestoreLock(rbdSnap.SnapID); err != nil {
			klog.Error(util.Log(ctx, err.Error()))
			return status.Error(codes.Aborted, err.Error())
		}
		defer cs.OperationLocks.ReleaseRestoreLock(rbdSnap.SnapID)

		err = cs.createVolumeFromSnapshot(ctx, cr, rbdVol, rbdSnap.SnapID)
		if err != nil {
			return err
		}
		util.DebugLog(ctx, "created volume %s from snapshot %s", rbdVol.RequestName, rbdSnap.RbdSnapName)
	} else if parentVol != nil {
		if err = cs.OperationLocks.GetCloneLock(parentVol.VolID); err != nil {
			klog.Error(util.Log(ctx, err.Error()))
			return status.Error(codes.Aborted, err.Error())
		}
		defer cs.OperationLocks.ReleaseCloneLock(parentVol.VolID)
		return rbdVol.createCloneFromImage(ctx, parentVol)
	} else {
		err = createImage(ctx, rbdVol, cr)
		if err != nil {
			klog.Errorf(util.Log(ctx, "failed to create volume: %v"), err)
			return status.Error(codes.Internal, err.Error())
		}
	}

	util.DebugLog(ctx, "created volume %s backed by image %s", rbdVol.RequestName, rbdVol.RbdImageName)

	defer func() {
		if err != nil {
			if !errors.Is(err, ErrFlattenInProgress) {
				if deleteErr := deleteImage(ctx, rbdVol, cr); deleteErr != nil {
					klog.Errorf(util.Log(ctx, "failed to delete rbd image: %s with error: %v"), rbdVol, deleteErr)
				}
			}
		}
	}()
	err = rbdVol.getImageID()
	if err != nil {
		klog.Errorf(util.Log(ctx, "failed to get volume id %s: %v"), rbdVol, err)
		return status.Error(codes.Internal, err.Error())
	}

	err = j.StoreImageID(ctx, rbdVol.JournalPool, rbdVol.ReservedID, rbdVol.ImageID, cr)
	if err != nil {
		klog.Errorf(util.Log(ctx, "failed to reserve volume %s: %v"), rbdVol, err)
		return status.Error(codes.Internal, err.Error())
	}

	if rbdSnap != nil {
		err = rbdVol.flattenRbdImage(ctx, cr, false, rbdHardMaxCloneDepth, rbdSoftMaxCloneDepth)
		if err != nil {
			klog.Errorf(util.Log(ctx, "failed to flatten image %s: %v"), rbdVol, err)
			return err
		}
	}
	if rbdVol.Encrypted {
		err = rbdVol.ensureEncryptionMetadataSet(rbdImageRequiresEncryption)
		if err != nil {
			klog.Errorf(util.Log(ctx, "failed to save encryption status, deleting image %s: %s"),
				rbdVol, err)
			return status.Error(codes.Internal, err.Error())
		}
	}
	return nil
}

```

###### 2.1 StoreImageID

调用setOMapKeys将ImageID与ReservedID存入omap中。

```
//ceph-csi/internal/journal/voljournal.go

// StoreImageID stores the image ID in omap.
func (conn *Connection) StoreImageID(ctx context.Context, pool, reservedUUID, imageID string, cr *util.Credentials) error {
	err := setOMapKeys(ctx, conn, pool, conn.config.namespace, conn.config.cephUUIDDirectoryPrefix+reservedUUID,
		map[string]string{conn.config.csiImageIDKey: imageID})
	if err != nil {
		return err
	}
	return nil
}

//ceph-csi/internal/journal/voljournal.go

// StoreImageID stores the image ID in omap.
func (conn *Connection) StoreImageID(ctx context.Context, pool, reservedUUID, imageID string, cr *util.Credentials) error {
	err := setOMapKeys(ctx, conn, pool, conn.config.namespace, conn.config.cephUUIDDirectoryPrefix+reservedUUID,
		map[string]string{conn.config.csiImageIDKey: imageID})
	if err != nil {
		return err
	}
	return nil
}

```

**（2）DeleteVolume**

简介

调用ceph存储后端，删除存储（rbd image）。

DeleteVolume deletes the volume in backend and removes the volume metadata from store



大致步骤：

（1）检查该driver是否支持存储的删除，不支持则直接返回错误；

（2）判断image是否还在使用（是否有watchers）；

（3）没被使用则调用ceph存储后端删除该image。



**DeleteVolume**

主体流程：

（1）检查该driver是否支持存储的删除，不支持则直接返回错误；

（2）根据secret构建ceph请求凭证（secret由external-provisioner组件传入）；

（3）从请求参数获取volumeID；

（4）将请求参数转换为rbdVol结构体；

（5）判断rbd image是否还在被使用，被使用则返回错误；

（6）删除temporary rbd image（clone操作的临时image）；

（7）删除rbd image。

```
//ceph-csi/internal/rbd/controllerserver.go

func (cs *ControllerServer) DeleteVolume(ctx context.Context, req *csi.DeleteVolumeRequest) (*csi.DeleteVolumeResponse, error) {
    // （1）校验请求参数；
	if err := cs.Driver.ValidateControllerServiceRequest(csi.ControllerServiceCapability_RPC_CREATE_DELETE_VOLUME); err != nil {
		klog.Errorf(util.Log(ctx, "invalid delete volume req: %v"), protosanitizer.StripSecrets(req))
		return nil, err
	}
    
    // （2）根据secret构建ceph请求凭证；
	cr, err := util.NewUserCredentials(req.GetSecrets())
	if err != nil {
		return nil, status.Error(codes.Internal, err.Error())
	}
	defer cr.DeleteCredentials()
    
    // （3）从请求参数获取volumeID； 
	// For now the image get unconditionally deleted, but here retention policy can be checked
	volumeID := req.GetVolumeId()
	if volumeID == "" {
		return nil, status.Error(codes.InvalidArgument, "empty volume ID in request")
	}

	if acquired := cs.VolumeLocks.TryAcquire(volumeID); !acquired {
		klog.Errorf(util.Log(ctx, util.VolumeOperationAlreadyExistsFmt), volumeID)
		return nil, status.Errorf(codes.Aborted, util.VolumeOperationAlreadyExistsFmt, volumeID)
	}
	defer cs.VolumeLocks.Release(volumeID)

	// lock out volumeID for clone and expand operation
	if err = cs.OperationLocks.GetDeleteLock(volumeID); err != nil {
		klog.Error(util.Log(ctx, err.Error()))
		return nil, status.Error(codes.Aborted, err.Error())
	}
	defer cs.OperationLocks.ReleaseDeleteLock(volumeID)

	rbdVol := &rbdVolume{}
	defer rbdVol.Destroy()
    
    // （4）将请求参数转换为rbdVol结构体； 
	rbdVol, err = genVolFromVolID(ctx, volumeID, cr, req.GetSecrets())
	if err != nil {
		if errors.Is(err, util.ErrPoolNotFound) {
			klog.Warningf(util.Log(ctx, "failed to get backend volume for %s: %v"), volumeID, err)
			return &csi.DeleteVolumeResponse{}, nil
		}

		// if error is ErrKeyNotFound, then a previous attempt at deletion was complete
		// or partially complete (image and imageOMap are garbage collected already), hence return
		// success as deletion is complete
		if errors.Is(err, util.ErrKeyNotFound) {
			klog.Warningf(util.Log(ctx, "Failed to volume options for %s: %v"), volumeID, err)
			return &csi.DeleteVolumeResponse{}, nil
		}

		// All errors other than ErrImageNotFound should return an error back to the caller
		if !errors.Is(err, ErrImageNotFound) {
			return nil, status.Error(codes.Internal, err.Error())
		}

		// If error is ErrImageNotFound then we failed to find the image, but found the imageOMap
		// to lead us to the image, hence the imageOMap needs to be garbage collected, by calling
		// unreserve for the same
		if acquired := cs.VolumeLocks.TryAcquire(rbdVol.RequestName); !acquired {
			klog.Errorf(util.Log(ctx, util.VolumeOperationAlreadyExistsFmt), rbdVol.RequestName)
			return nil, status.Errorf(codes.Aborted, util.VolumeOperationAlreadyExistsFmt, rbdVol.RequestName)
		}
		defer cs.VolumeLocks.Release(rbdVol.RequestName)

		if err = undoVolReservation(ctx, rbdVol, cr); err != nil {
			return nil, status.Error(codes.Internal, err.Error())
		}
		return &csi.DeleteVolumeResponse{}, nil
	}
	defer rbdVol.Destroy()

	// lock out parallel create requests against the same volume name as we
	// cleanup the image and associated omaps for the same
	if acquired := cs.VolumeLocks.TryAcquire(rbdVol.RequestName); !acquired {
		klog.Errorf(util.Log(ctx, util.VolumeOperationAlreadyExistsFmt), rbdVol.RequestName)
		return nil, status.Errorf(codes.Aborted, util.VolumeOperationAlreadyExistsFmt, rbdVol.RequestName)
	}
	defer cs.VolumeLocks.Release(rbdVol.RequestName)
    
    // （5）判断rbd image是否还在被使用，被使用则返回错误；
	inUse, err := rbdVol.isInUse()
	if err != nil {
		klog.Errorf(util.Log(ctx, "failed getting information for image (%s): (%s)"), rbdVol, err)
		return nil, status.Error(codes.Internal, err.Error())
	}
	if inUse {
		klog.Errorf(util.Log(ctx, "rbd %s is still being used"), rbdVol)
		return nil, status.Errorf(codes.Internal, "rbd %s is still being used", rbdVol.RbdImageName)
	}
    
    // （6）删除temporary rbd image；
	// delete the temporary rbd image created as part of volume clone during
	// create volume
	tempClone := rbdVol.generateTempClone()
	err = deleteImage(ctx, tempClone, cr)
	if err != nil {
		// return error if it is not ErrImageNotFound
		if !errors.Is(err, ErrImageNotFound) {
			klog.Errorf(util.Log(ctx, "failed to delete rbd image: %s with error: %v"),
				tempClone, err)
			return nil, status.Error(codes.Internal, err.Error())
		}
	}

    // （7）删除rbd image.
	// Deleting rbd image
	util.DebugLog(ctx, "deleting image %s", rbdVol.RbdImageName)
	if err = deleteImage(ctx, rbdVol, cr); err != nil {
		klog.Errorf(util.Log(ctx, "failed to delete rbd image: %s with error: %v"),
			rbdVol, err)
		return nil, status.Error(codes.Internal, err.Error())
	}

	if err = undoVolReservation(ctx, rbdVol, cr); err != nil {
		klog.Errorf(util.Log(ctx, "failed to remove reservation for volume (%s) with backing image (%s) (%s)"),
			rbdVol.RequestName, rbdVol.RbdImageName, err)
		return nil, status.Error(codes.Internal, err.Error())
	}

	if rbdVol.Encrypted {
		if err = rbdVol.KMS.DeletePassphrase(rbdVol.VolID); err != nil {
			klog.Warningf(util.Log(ctx, "failed to clean the passphrase for volume %s: %s"), rbdVol.VolID, err)
		}
	}

	return &csi.DeleteVolumeResponse{}, nil
}

```

###### isInUse

通过列出image的watcher来判断该image是否在被使用。当image没有watcher时，isInUse返回false，否则返回true。

```
//ceph-csi/internal/rbd/rbd_util.go

func (rv *rbdVolume) isInUse() (bool, error) {
	image, err := rv.open()
	if err != nil {
		if errors.Is(err, ErrImageNotFound) || errors.Is(err, util.ErrPoolNotFound) {
			return false, err
		}
		// any error should assume something else is using the image
		return true, err
	}
	defer image.Close()

	watchers, err := image.ListWatchers()
	if err != nil {
		return false, err
	}

	// because we opened the image, there is at least one watcher
	return len(watchers) != 1, nil
}

```

**（3）ControllerExpandVolume**

**简介**

调用ceph存储后端，扩容存储（rbd image）。



ControllerExpandVolume expand RBD Volumes on demand based on resizer request.



大致步骤：

（1）检查该driver是否支持存储扩容，不支持则直接返回错误；

（2）调用ceph存储后端扩容该image。



实际上，存储扩容分为两大步骤，第一步是external-provisioner组件调用rbdType-ControllerServer类型的ceph-csi进行ControllerExpandVolume，主要负责将底层存储扩容；第二步是kubelet调用rbdType-NodeServer类型的ceph-csi进行NodeExpandVolume，主要负责将底层rbd image的扩容信息同步到rbd/nbd device，对xfs/ext文件系统进行扩展（当volumemode是block，则不用进行node端扩容操作）。



**ControllerExpandVolume**

主体流程：

（1）检查该driver是否支持存储扩容，不支持则直接返回错误；

（2）根据secret构建ceph请求凭证（secret由external-provisioner组件传入）；

（3）将请求参数转换为rbdVol结构体；

（4）调用rbdVol.resize去扩容image。

```
func (cs *ControllerServer) ControllerExpandVolume(ctx context.Context, req *csi.ControllerExpandVolumeRequest) (*csi.ControllerExpandVolumeResponse, error) {
	if err := cs.Driver.ValidateControllerServiceRequest(csi.ControllerServiceCapability_RPC_EXPAND_VOLUME); err != nil {
		klog.Errorf(util.Log(ctx, "invalid expand volume req: %v"), protosanitizer.StripSecrets(req))
		return nil, err
	}

	volID := req.GetVolumeId()
	if volID == "" {
		return nil, status.Error(codes.InvalidArgument, "volume ID cannot be empty")
	}

	capRange := req.GetCapacityRange()
	if capRange == nil {
		return nil, status.Error(codes.InvalidArgument, "capacityRange cannot be empty")
	}

	// lock out parallel requests against the same volume ID
	if acquired := cs.VolumeLocks.TryAcquire(volID); !acquired {
		klog.Errorf(util.Log(ctx, util.VolumeOperationAlreadyExistsFmt), volID)
		return nil, status.Errorf(codes.Aborted, util.VolumeOperationAlreadyExistsFmt, volID)
	}
	defer cs.VolumeLocks.Release(volID)

	// lock out volumeID for clone and delete operation
	if err := cs.OperationLocks.GetExpandLock(volID); err != nil {
		klog.Error(util.Log(ctx, err.Error()))
		return nil, status.Error(codes.Aborted, err.Error())
	}
	defer cs.OperationLocks.ReleaseExpandLock(volID)

	cr, err := util.NewUserCredentials(req.GetSecrets())
	if err != nil {
		return nil, status.Error(codes.Internal, err.Error())
	}
	defer cr.DeleteCredentials()

	rbdVol := &rbdVolume{}
	defer rbdVol.Destroy()

	rbdVol, err = genVolFromVolID(ctx, volID, cr, req.GetSecrets())
	if err != nil {
		// nolint:gocritic // this ifElseChain can not be rewritten to a switch statement
		if errors.Is(err, ErrImageNotFound) {
			err = status.Errorf(codes.NotFound, "volume ID %s not found", volID)
		} else if errors.Is(err, util.ErrPoolNotFound) {
			klog.Errorf(util.Log(ctx, "failed to get backend volume for %s: %v"), volID, err)
			err = status.Errorf(codes.NotFound, err.Error())
		} else {
			err = status.Errorf(codes.Internal, err.Error())
		}
		return nil, err
	}
	defer rbdVol.Destroy()

	if rbdVol.Encrypted {
		return nil, status.Errorf(codes.InvalidArgument, "encrypted volumes do not support resize (%s)",
			rbdVol)
	}

	// always round up the request size in bytes to the nearest MiB/GiB
	volSize := util.RoundOffBytes(req.GetCapacityRange().GetRequiredBytes())

	// resize volume if required
	nodeExpansion := false
	if rbdVol.VolSize < volSize {
		util.DebugLog(ctx, "rbd volume %s size is %v,resizing to %v", rbdVol, rbdVol.VolSize, volSize)
		rbdVol.VolSize = volSize
		nodeExpansion = true
		err = rbdVol.resize(ctx, cr)
		if err != nil {
			klog.Errorf(util.Log(ctx, "failed to resize rbd image: %s with error: %v"), rbdVol, err)
			return nil, status.Error(codes.Internal, err.Error())
		}
	}

	return &csi.ControllerExpandVolumeResponse{
		CapacityBytes:         rbdVol.VolSize,
		NodeExpansionRequired: nodeExpansion,
	}, nil
}

```

###### rbdVol.resize

拼接resize命令并执行。

```
func (rv *rbdVolume) resize(ctx context.Context, cr *util.Credentials) error {
	mon := rv.Monitors
	volSzMiB := fmt.Sprintf("%dM", util.RoundOffVolSize(rv.VolSize))

	args := []string{"resize", rv.String(), "--size", volSzMiB, "--id", cr.ID, "-m", mon, "--keyfile=" + cr.KeyFile}
	_, stderr, err := util.ExecCommand(ctx, "rbd", args...)

	if err != nil {
		return fmt.Errorf("failed to resize rbd image (%w), command output: %s", err, stderr)
	}

	return nil
}

```

至此，rbd driver-controllerserver的分析已经完成，下面做个总结。



rbd driver-controllerserver分析总结

（1）controllerserver主要包括了CreateVolume（创建存储）、DeleteVolume（删除存储）、ControllerExpandVolume（存储扩容）、CreateSnapshot（创建存储快照）、DeleteSnapshot（删除存储快照）操作，这里主要对CreateVolume（创建存储）、DeleteVolume（删除存储）、ControllerExpandVolume（存储扩容）进行分析，作用分别如下：



CreateVolume：调用ceph存储后端，创建存储（rbd image）。

DeleteVolume：调用ceph存储后端，删除存储（rbd image）。

ControllerExpandVolume：调用ceph存储后端，扩容存储（rbd image）。

（2）存储扩容分为两大步骤，第一步是external-provisioner组件调用rbdType-ControllerServer类型的ceph-csi进行ControllerExpandVolume，主要负责将底层存储扩容；第二步是kubelet调用rbdType-NodeServer类型的ceph-csi进行NodeExpandVolume，主要负责将底层rbd image的扩容信息同步到rbd/nbd device，对xfs/ext文件系统进行扩展（当volumemode是block，则不用进行node端扩容操作）



