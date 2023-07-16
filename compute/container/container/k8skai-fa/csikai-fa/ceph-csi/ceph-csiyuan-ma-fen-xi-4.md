## ceph-csi组件源码分析（1）-组件介绍与部署[yaml](https://so.csdn.net/so/search?q=yaml&spm=1001.2101.3001.7020)分析

## 

概述

ceph-csi扩展各种存储类型的卷的管理能力，实现第三方存储ceph的各种操作能力与k8s存储系统的结合。调用第三方存储ceph的接口或命令，从而提供ceph数据卷的创建/删除、挂载/解除挂载的具体操作实现。前面分析组件中的对于数据卷的创建/删除、挂载/解除挂载操作，全是调用ceph-csi，然后由ceph-csi调用ceph提供的命令或接口来完成最终的操作。



ceph-csi组件的源码分析分为五部分：

（1）组件介绍与部署yaml分析；

（2）组件启动参数分析；

（3）rbd driver分析；

（4）cephfs driver分析；

（5）liveness driver分析。



本节先进行组件介绍与部署yaml分析。



ceph-csi组件-作用介绍

（1）create pvc时，external-provisioner组件监听到pvc创建事件后，负责拼接请求，然后调用ceph-csi的CreateVolume方法来创建存储；



（2）delete pvc时，pv对象状态由bound变为release，external-provisioner监听到pv更新事件后，负责拼接请求，调用ceph-csi的DeleteVolume方法来删除存储。



（3）create pod cliam pvc时，kubelet会调用ceph-csi组件将创建好的存储从ceph集群挂载到pod所在的node上，然后再挂载到pod相应的目录上；



（4）delete pod cliam pvc时，kubelet会调用ceph-csi组件相应方法，解除存储在pod目录上的挂载，再解除存储在node上的挂载。



ceph-csi组件-服务组成

ceph-csi含有rbdType、cephfsType、livenessType三大类型服务，可以通过启动参数指定一种服务来进行启动。



而rbdType、cephfsType类型的服务可以继续细分，包括了NodeServer、ControllerServer与IdentityServer三种具体的服务，其中NodeServer与ControllerServer只能选其一进行启动，IdentityServer会伴随着NodeServer或ControllerServer的启动而启动。

![](/assets/compute-container-k8s-cephcsi141.png)

后面会详细分析每一种服务中的方法。



**rbdTy**pe

该类型服务包含了rbd的相关操作。



**cephfsType**

该类型服务包含了cephfs的相关操作。



**livenessType**

该类型服务主要是定时向csi endpoint探测csi组件的存活（向指定的socket地址发送probe请求），然后统计到prometheus指标中。



**三种具体服务**

**NodeServer**

部署在k8s中的每个node上，主要负责cephfs、rbd在node节点上相关的操作，如将存储挂载到node上，解除node上存储挂载等操作。



**ControllerServer**

主要负责创建、删除cephfs/rbd存储等操作。



**IdentityServer**

主要是返回自身服务的相关信息，如返回服务身份信息（名称与版本等信息）、返回服务具备的能力、暴露存活探测接口（用于给别的组件/服务探测该服务是否存活）等。



**ceph-csi组件源码分析**

该组件的源码分析将按照以下顺序进行：

（1）ceph-csi及相关组件部署yaml分析

（2）main函数分析

（3）rbdType/cephfsType/livenessType

（4）NodeServer/ControllerServer/IdentityServer



本文先进行（1）ceph-csi及相关组件部署yaml分析与（2）main函数的分析。



**1.ceph-csi及相关组件部署yaml分析**

**下面对部署yaml的分析，以rbd为例进行分析，cephfs与rbd类似。**

部署yaml请参考：

https://github.com/ceph/ceph-csi/blob/devel/deploy/rbd/kubernetes/csi-rbdplugin-provisioner.yaml

https://github.com/ceph/ceph-csi/blob/devel/deploy/rbd/kubernetes/csi-rbdplugin.yaml



其中csi-rbdplugin-provisioner.yaml包含了名称为csi-rbdplugin-provisioner的deployment，csi-rbdplugin.yaml包含了名称为csi-rbdplugin的daemonset。



**deployment：csi-rbdplugin-provisioner**

包含了csi-provisioner、csi-snapshotter、csi-attacher、csi-resizer、csi-rbdplugin、liveness-prometheus5个容器，作用分别如下：

（1）csi-provisioner：实际上是external-provisioner组件，前面已经做过详细介绍，这里再来简单回顾一下。create pvc时，csi-provisioner参与存储资源与pv对象的创建。csi-provisioner组件监听到pvc创建事件后，负责拼接请求，调用ceph-csi组件（即csi-rbdplugin容器）的CreateVolume方法来创建存储，创建存储成功后，创建pv对象；delete pvc时，csi-provisioner参与存储资源与pv对象的删除。当pvc被删除时，pv controller会将其绑定的pv对象状态由bound更新为release，csi-provisioner监听到pv更新事件后，调用ceph-csi组件（即csi-rbdplugin容器）的DeleteVolume方法来删除存储，并删除pv对象。



（2）csi-snapshotter：实际上是external-snapshotter组件，负责处理存储快照相关的操作，后面再做详细介绍。



（3）csi-attacher：实际上是external-attacher组件，只负责操作VolumeAttachment对象，实际上并没有操作存储，后面再做详细介绍。



（4）csi-resizer：实际上是external-resizer组件，负责处理存储扩容相关的操作，后面再做详细介绍。



（5）csi-rbdplugin：实际上是ceph-csi组件，rbdType-ControllerServer/IdentityServer类型的服务。create pvc时，external-provisioner组件（即csi-provisioner容器）监听到pvc创建事件后，负责拼接请求，然后调用csi-rbdplugin容器的CreateVolume方法来创建存储；delete pvc时，pv对象状态由bound变为release，external-provisioner组件（即csi-provisioner容器）监听到pv更新事件后，负责拼接请求，调用csi-rbdplugin容器的DeleteVolume方法来删除存储。



（6）liveness-prometheus：实际上是ceph-csi组件，livenessType类型的服务。负责探测并上报csi-rbdplugin服务的存活情况。



**daemonset：csi-rbdplugin**

包含了driver-registrar、csi-rbdplugin、liveness-prometheus 3个容器，作用分别如下：

（1）driver-registrar：向kubelet传入csi-rbdplugin容器提供服务的socket地址、版本信息和驱动名称（如rbd.csi.ceph.com）等，将csi-rbdplugin容器服务注册给kubelet，后面再做详细介绍。



（2）csi-rbdplugin：实际上是ceph-csi组件，rbdType-NoderServer/IdentityServer类型的服务。create pod cliam pvc时，kubelet会调用csi-rbdplugin容器将创建好的存储从ceph集群挂载到pod所在的node上，然后再挂载到pod相应的目录上；delete pod cliam pvc时，kubelet会调用csi-rbdplugin容器的相应方法，解除存储在pod目录上的挂载，再解除存储在node上的挂载。



（3）liveness-prometheus：实际上是ceph-csi组件，livenessType类型的服务。负责探测并上报csi-rbdplugin服务的存活情况。



**2.main函数的分析**

主要逻辑：

（1）判断–version参数是否为true，是则输出版本信息并退出；

（2）校验driver type是否指定；

（3）获取driver名称并校验；

（4）PidLimit相关设置；

（5）当driver type为liveness或者开启Metrics时，设置pod ip并校验Metrics url；

（6）根据不同的driver type（rbdType/cephfsType/livenessType）调用不同类型的服务的run方法，来启动不同类型的服务。

```
func main() {
    // --version参数输出
	if conf.Version {
		fmt.Println("Cephcsi Version:", util.DriverVersion)
		fmt.Println("Git Commit:", util.GitCommit)
		fmt.Println("Go Version:", runtime.Version())
		fmt.Println("Compiler:", runtime.Compiler)
		fmt.Printf("Platform: %s/%s\n", runtime.GOOS, runtime.GOARCH)
		if kv, err := util.GetKernelVersion(); err == nil {
			fmt.Println("Kernel:", kv)
		}
		os.Exit(0)
	}
	util.DefaultLog("Driver version: %s and Git version: %s", util.DriverVersion, util.GitCommit)
    
    // 校验driver type是否指定
	if conf.Vtype == "" {
		klog.Fatalln("driver type not specified")
	}
    
    // 获取driver名称并校验
	dname := getDriverName()
	err := util.ValidateDriverName(dname)
	if err != nil {
		klog.Fatalln(err) // calls exit
	}  
	
    // PidLimit相关设置
	// the driver may need a higher PID limit for handling all concurrent requests
	if conf.PidLimit != 0 {
		currentLimit, pidErr := util.GetPIDLimit()
		if pidErr != nil {
			klog.Errorf("Failed to get the PID limit, can not reconfigure: %v", pidErr)
		} else {
			util.DefaultLog("Initial PID limit is set to %d", currentLimit)
			err = util.SetPIDLimit(conf.PidLimit)
			if err != nil {
				klog.Errorf("Failed to set new PID limit to %d: %v", conf.PidLimit, err)
			} else {
				s := ""
				if conf.PidLimit == -1 {
					s = " (max)"
				}
				util.DefaultLog("Reconfigured PID limit to %d%s", conf.PidLimit, s)
			}
		}
	}
    
    // 当driver type为liveness或者开启Metrics时，设置pod ip并校验Metrics url
	if conf.EnableGRPCMetrics || conf.Vtype == livenessType {
		// validate metrics endpoint
		conf.MetricsIP = os.Getenv("POD_IP")

		if conf.MetricsIP == "" {
			klog.Warning("missing POD_IP env var defaulting to 0.0.0.0")
			conf.MetricsIP = "0.0.0.0"
		}
		err = util.ValidateURL(&conf)
		if err != nil {
			klog.Fatalln(err)
		}
	}
    
    // 根据不同的driver type调用不同的服务的run方法，来启动不同的服务
	util.DefaultLog("Starting driver type: %v with name: %v", conf.Vtype, dname)
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

	os.Exit(0)
}

```



