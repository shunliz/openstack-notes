# ceph-csi源码分析（8）-cephfs driver分析 {#articleContentId}

当ceph-csi组件启动时指定的driver type为cephfs时，会启动cephfs driver相关的服务。然后再根据controllerserver、nodeserver的参数配置，决定启动ControllerServer与IdentityServer，或NodeServer与IdentityServer。

基于tag v3.0.0

[https://github.com/ceph/ceph-csi/releases/tag/v3.0.0](https://github.com/ceph/ceph-csi/releases/tag/v3.0.0)

cephfs driver，与rbd driver类似，同样包括了controllerserver、nodeserver与IdentityServer，且大部分方法逻辑一致，只是最后调用的cli命令稍有不同，所以大部分方法的分析可以参考rbd driver部分。

![](/assets/compute-container-k8s-cephcsi13633361.png)



其中，controllerserver主要包括了CreateVolume（创建存储）、DeleteVolume（删除存储）、ControllerExpandVolume（存储扩容）：



CreateVolume：调用ceph存储后端，创建存储（与rbd逻辑类似，不过cephfs这里创建的是目录，而不是rbd image）。

DeleteVolume：调用ceph存储后端，删除存储（与rbd逻辑类似，不过cephfs这里删除的是目录，而不是rbd image）。

ControllerExpandVolume：调用ceph存储后端，扩容存储（重新设置cephfs目录的quota）。

nodeserver主要包括了NodeGetCapabilities（获取driver能力）、NodeGetVolumeStats（存储探测及metrics获取）、NodeStageVolume（mount stagingPath）、NodePublishVolume（mount targetPath）、NodeUnpublishVolume（umount targetPath）、NodeUnstageVolume（umount stagingPath）：



NodeGetCapabilities：获取ceph-csi driver的能力。

NodeGetVolumeStats：探测挂载存储的状态，并返回该存储的相关metrics给kubelet。

NodeStageVolume：将cephfs的远端目录挂载到node上的staging path。

NodePublishVolume：将NodeStageVolume方法中的staging path，mount到target path。

NodeUnpublishVolume：解除掉stagingPath到targetPath的挂载。

NodeUnstageVolume：将cephfs的远端目录到node上的staging path的挂载解除掉。

IdentityServer主要包括了GetPluginInfo（获取driver信息）、Probe（探测接口）、GetPluginCapabilities（获取driver能力）三个方法：



GetPluginInfo：用于获取该ceph-csi driver的信息，如driver名称、版本等。

Probe：一个探测接口，用于探测该driver是否启动。

GetPluginCapabilities：用于获取driver的能力。



**cephfs挂载知识讲解**

cephfs挂载分为fuse挂载和内核挂载。



一个cephfs存储挂载给pod，一共分为2个步骤，分别如下：



1.kubelet组件调用cephfsType-nodeserver-ceph-csi的NodeStageVolume方法，将cephfs的远端目录挂载到node上的staging path；



2.kubelet组件调用cephfsType-nodeserver-ceph-csi的NodePublishVolume方法，将上一步骤中的staging path mount到target path。



可以看出，与rbd image挂载给pod相比，在NodeStageVolume方法中少了一个map rbd/nbd device的操作，同样的，在NodeUnstageVolume方法中也会少一个unmap rbd/nbd device的操作。



**cephfs解除挂载知识讲解**

一个cephfs存储从pod中解除挂载，一共分为2个步骤，分别如下：



1.kubelet组件调用cephfsType-nodeserver-ceph-csi的NodeUnpublishVolume方法，解除掉stagingPath与targetPath的挂载关系。



2.kubelet组件调用cephfsType-nodeserver-ceph-csi的NodeUnstageVolume方法，先解除掉targetPath到远端cephfs存储（目录）的挂载关系。



cephfs存储挂载给pod后，node上会出现2个mount关系，示例如下：



\# mount \| grep ceph-fuse

ceph-fuse on /home/cld/kubernetes/lib/kubelet/plugins/kubernetes.io/csi/pv/pvc-fa752c51-79d4-42f2-a3ff-9d7afe8767b5/globalmount type fuse.ceph-fuse \(rw,nosuid,nodev,relatime,user\_id=0,group\_id=0,allow\_other\)

ceph-fuse on /home/cld/kubernetes/lib/kubelet/pods/87f7e220-8b2d-4cd3-8395-12794940fa2e/volumes/kubernetes.io~csi/pvc-fa752c51-79d4-42f2-a3ff-9d7afe8767b5/mount type fuse.ceph-fuse \(rw,nosuid,nodev,relatime,user\_id=0,group\_id=0,allow\_other,\_netdev\)



其中/home/cld/kubernetes/lib/kubelet/plugins/kubernetes.io/csi/pv/pvc-fa752c51-79d4-42f2-a3ff-9d7afe8767b5/globalmount为staging path；而/home/cld/kubernetes/lib/kubelet/pods/87f7e220-8b2d-4cd3-8395-12794940fa2e/volumes/kubernetes.io~csi/pvc-fa752c51-79d4-42f2-a3ff-9d7afe8767b5/mount为target path。



**cephfs driver分析**

下面将对cephfs driver中与rbd driver不一致的展开分析。



**（1）cephfs扩容逻辑**

cephfs driver没有NodeExpandVolume（node端存储扩容），与rbd存储扩容分为两大步骤不一样，cephfs存储扩容只需一步，就是csi的ControllerExpandVolume，主要负责将cephfs存储扩容（即重新设置cephfs目录的quota）。



cephfs driver的NodeGetCapabilities方法中，相比于rbd driver，也少了node端存储扩容的能力。

```

```

**（2）NodeStageVolume**

将cephfs的远端目录挂载到node上的staging path。



NodeStageVolume

主要逻辑：

（1）校验请求参数；

（2）构建volOptions；

（3）检查stagingPath是否是挂载点；

（4）调用ns.mount进行挂载操作。



###### ns.mount

这里的挂载分为fuse挂载和内核挂载，不同的挂载调用不同的本地command来进行。

```
// internal/cephfs/nodeserver.go
func (*NodeServer) mount(ctx context.Context, volOptions *volumeOptions, req *csi.NodeStageVolumeRequest) error {
	stagingTargetPath := req.GetStagingTargetPath()
	volID := volumeID(req.GetVolumeId())

	cr, err := getCredentialsForVolume(volOptions, req)
	if err != nil {
		klog.Errorf(util.Log(ctx, "failed to get ceph credentials for volume %s: %v"), volID, err)
		return status.Error(codes.Internal, err.Error())
	}
	defer cr.DeleteCredentials()

	m, err := newMounter(volOptions)
	if err != nil {
		klog.Errorf(util.Log(ctx, "failed to create mounter for volume %s: %v"), volID, err)
		return status.Error(codes.Internal, err.Error())
	}

	util.DebugLog(ctx, "cephfs: mounting volume %s with %s", volID, m.name())

	readOnly := "ro"
	fuseMountOptions := strings.Split(volOptions.FuseMountOptions, ",")
	kernelMountOptions := strings.Split(volOptions.KernelMountOptions, ",")

	if req.VolumeCapability.AccessMode.Mode == csi.VolumeCapability_AccessMode_MULTI_NODE_READER_ONLY ||
		req.VolumeCapability.AccessMode.Mode == csi.VolumeCapability_AccessMode_SINGLE_NODE_READER_ONLY {
		switch m.(type) {
		case *fuseMounter:
			if !csicommon.MountOptionContains(strings.Split(volOptions.FuseMountOptions, ","), readOnly) {
				volOptions.FuseMountOptions = util.MountOptionsAdd(volOptions.FuseMountOptions, readOnly)
				fuseMountOptions = append(fuseMountOptions, readOnly)
			}
		case *kernelMounter:
			if !csicommon.MountOptionContains(strings.Split(volOptions.KernelMountOptions, ","), readOnly) {
				volOptions.KernelMountOptions = util.MountOptionsAdd(volOptions.KernelMountOptions, readOnly)
				kernelMountOptions = append(kernelMountOptions, readOnly)
			}
		}
	}

	if err = m.mount(ctx, stagingTargetPath, cr, volOptions); err != nil {
		klog.Errorf(util.Log(ctx,
			"failed to mount volume %s: %v Check dmesg logs if required."),
			volID,
			err)
		return status.Error(codes.Internal, err.Error())
	}
	if !csicommon.MountOptionContains(kernelMountOptions, readOnly) && !csicommon.MountOptionContains(fuseMountOptions, readOnly) {
		// #nosec - allow anyone to write inside the stagingtarget path
		err = os.Chmod(stagingTargetPath, 0777)
		if err != nil {
			klog.Errorf(util.Log(ctx, "failed to change stagingtarget path %s permission for volume %s: %v"), stagingTargetPath, volID, err)
			uErr := unmountVolume(ctx, stagingTargetPath)
			if uErr != nil {
				klog.Errorf(util.Log(ctx, "failed to umount stagingtarget path %s for volume %s: %v"), stagingTargetPath, volID, uErr)
			}
			return status.Error(codes.Internal, err.Error())
		}
	}
	return nil
}

```



**fuse挂载**

```
// internal/cephfs/volumemounter.go 
func (m *fuseMounter) mount(ctx context.Context, mountPoint string, cr *util.Credentials, volOptions *volumeOptions) error {
	if err := util.CreateMountPoint(mountPoint); err != nil {
		return err
	}

	return mountFuse(ctx, mountPoint, cr, volOptions)
}

func mountFuse(ctx context.Context, mountPoint string, cr *util.Credentials, volOptions *volumeOptions) error {
	args := []string{
		mountPoint,
		"-m", volOptions.Monitors,
		"-c", util.CephConfigPath,
		"-n", cephEntityClientPrefix + cr.ID, "--keyfile=" + cr.KeyFile,
		"-r", volOptions.RootPath,
		"-o", "nonempty",
	}

	if volOptions.FuseMountOptions != "" {
		args = append(args, ","+volOptions.FuseMountOptions)
	}

	if volOptions.FsName != "" {
		args = append(args, "--client_mds_namespace="+volOptions.FsName)
	}

	_, stderr, err := util.ExecCommand(ctx, "ceph-fuse", args[:]...)
	if err != nil {
		return err
	}

	// Parse the output:
	// We need "starting fuse" meaning the mount is ok
	// and PID of the ceph-fuse daemon for unmount

	match := fusePidRx.FindSubmatch([]byte(stderr))
	// validMatchLength is set to 2 as match is expected
	// to have 2 items, starting fuse and PID of the fuse daemon
	const validMatchLength = 2
	if len(match) != validMatchLength {
		return fmt.Errorf("ceph-fuse failed: %s", stderr)
	}

	pid, err := strconv.Atoi(string(match[1]))
	if err != nil {
		return fmt.Errorf("failed to parse FUSE daemon PID: %w", err)
	}

	fusePidMapMtx.Lock()
	fusePidMap[mountPoint] = pid
	fusePidMapMtx.Unlock()

	return nil
}

```

**内核挂载**

```
// internal/cephfs/volumemounter.go 
func (m *kernelMounter) mount(ctx context.Context, mountPoint string, cr *util.Credentials, volOptions *volumeOptions) error {
	if err := util.CreateMountPoint(mountPoint); err != nil {
		return err
	}

	return mountKernel(ctx, mountPoint, cr, volOptions)
}

func mountKernel(ctx context.Context, mountPoint string, cr *util.Credentials, volOptions *volumeOptions) error {
	if err := execCommandErr(ctx, "modprobe", "ceph"); err != nil {
		return err
	}

	args := []string{
		"-t", "ceph",
		fmt.Sprintf("%s:%s", volOptions.Monitors, volOptions.RootPath),
		mountPoint,
	}

	optionsStr := fmt.Sprintf("name=%s,secretfile=%s", cr.ID, cr.KeyFile)
	mdsNamespace := ""
	if volOptions.FsName != "" {
		mdsNamespace = fmt.Sprintf("mds_namespace=%s", volOptions.FsName)
	}
	optionsStr = util.MountOptionsAdd(optionsStr, mdsNamespace, volOptions.KernelMountOptions, netDev)

	args = append(args, "-o", optionsStr)

	return execCommandErr(ctx, "mount", args[:]...)
}

```

**（3）NodeUnstageVolume**

将cephfs的远端目录到node上的staging path的挂载解除掉。



**NodeUnstageVolume**

主要逻辑：

（1）校验请求参数；

（2）调用unmountVolume解除cephfs的远端目录到node上的staging path的挂载。





