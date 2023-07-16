## [ceph](https://so.csdn.net/so/search?q=ceph&spm=1001.2101.3001.7020)-csi源码分析（2）-组件启动参数分析



ceph-csi组件启动参数列表

rbd驱动参数列表参考：https://github.com/ceph/ceph-csi/blob/devel/docs/deploy-rbd.md

cephfs驱动参数列表参考：https://github.com/ceph/ceph-csi/blob/devel/docs/deploy-cephfs.md



下面结合ceph-csi源码对组件启动参数进行分析。

```
const (
	rbdType      = "rbd"
	cephfsType   = "cephfs"
	livenessType = "liveness"

	rbdDefaultName      = "rbd.csi.ceph.com"
	cephfsDefaultName   = "cephfs.csi.ceph.com"
	livenessDefaultName = "liveness.csi.ceph.com"

	pollTime     = 60 // seconds
	probeTimeout = 3  // seconds
)

var (
	conf util.Config
)

func init() {
	// common flags
	flag.StringVar(&conf.Vtype, "type", "", "driver type [rbd|cephfs|liveness]")
	flag.StringVar(&conf.Endpoint, "endpoint", "unix://tmp/csi.sock", "CSI endpoint")
	flag.StringVar(&conf.DriverName, "drivername", "", "name of the driver")
	flag.StringVar(&conf.NodeID, "nodeid", "", "node id")
	flag.StringVar(&conf.InstanceID, "instanceid", "", "Unique ID distinguishing this instance of Ceph CSI among other"+
		" instances, when sharing Ceph clusters across CSI instances for provisioning")
	flag.IntVar(&conf.PidLimit, "pidlimit", 0, "the PID limit to configure through cgroups")
	flag.BoolVar(&conf.IsControllerServer, "controllerserver", false, "start cephcsi controller server")
	flag.BoolVar(&conf.IsNodeServer, "nodeserver", false, "start cephcsi node server")
	flag.StringVar(&conf.DomainLabels, "domainlabels", "", "list of kubernetes node labels, that determines the topology"+
		" domain the node belongs to, separated by ','")

	// cephfs related flags
	flag.BoolVar(&conf.ForceKernelCephFS, "forcecephkernelclient", false, "enable Ceph Kernel clients on kernel < 4.17 which support quotas")

	// liveness/grpc metrics related flags
	flag.IntVar(&conf.MetricsPort, "metricsport", 8080, "TCP port for liveness/grpc metrics requests")
	flag.StringVar(&conf.MetricsPath, "metricspath", "/metrics", "path of prometheus endpoint where metrics will be available")
	flag.DurationVar(&conf.PollTime, "polltime", time.Second*pollTime, "time interval in seconds between each poll")
	flag.DurationVar(&conf.PoolTimeout, "timeout", time.Second*probeTimeout, "probe timeout in seconds")

	flag.BoolVar(&conf.EnableGRPCMetrics, "enablegrpcmetrics", false, "[DEPRECATED] enable grpc metrics")
	flag.StringVar(&conf.HistogramOption, "histogramoption", "0.5,2,6",
		"[DEPRECATED] Histogram option for grpc metrics, should be comma separated value, ex:= 0.5,2,6 where start=0.5 factor=2, count=6")

	flag.UintVar(&conf.RbdHardMaxCloneDepth, "rbdhardmaxclonedepth", 8, "Hard limit for maximum number of nested volume clones that are taken before a flatten occurs")
	flag.UintVar(&conf.RbdSoftMaxCloneDepth, "rbdsoftmaxclonedepth", 4, "Soft limit for maximum number of nested volume clones that are taken before a flatten occurs")
	flag.UintVar(&conf.MaxSnapshotsOnImage, "maxsnapshotsonimage", 450, "Maximum number of snapshots allowed on rbd image without flattening")
	flag.BoolVar(&conf.SkipForceFlatten, "skipforceflatten", false,
		"skip image flattening if kernel support mapping of rbd images which has the deep-flatten feature")

	flag.BoolVar(&conf.Version, "version", false, "Print cephcsi version information")

	klog.InitFlags(nil)
	if err := flag.Set("logtostderr", "true"); err != nil {
		klog.Exitf("failed to set logtostderr flag: %v", err)
	}
	flag.Parse()
}

```

#### deployment：csi-rbdplugin容器部署的启动参数配置

deployment：csi-rbdplugin容器实际上是rbdType-ControllerServer服务，主要负责创建、删除rbd存储等操作。

```
          args:
            - "--nodeid=$(NODE_ID)"
            - "--type=rbd"
            - "--controllerserver=true"
            - "--endpoint=$(CSI_ENDPOINT)"
            - "--v=5"
            - "--drivername=rbd.csi.ceph.com"
            - "--pidlimit=-1"
            - "--rbdhardmaxclonedepth=8"
            - "--rbdsoftmaxclonedepth=4"
          env:
            - name: NODE_ID
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            - name: CSI_ENDPOINT
              value: unix:///csi/csi-provisioner.sock

```

#### daemonset：csi-rbdplugin容器部署的启动参数配置

daemonset：csi-rbdplugin容器实际上是rbdType-NodeServer服务，主要负责rbd存储的挂载、解除挂载等操作。

```
          args:
            - "--nodeid=$(NODE_ID)"
            - "--type=rbd"
            - "--nodeserver=true"
            - "--endpoint=$(CSI_ENDPOINT)"
            - "--v=5"
            - "--drivername=rbd.csi.ceph.com"
          env:
            - name: NODE_ID
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            - name: CSI_ENDPOINT
              value: unix:///csi/csi.sock

```

#### deployment：liveness-prometheus容器部署的启动参数配置

```
          args:
            - "--type=liveness"
            - "--endpoint=$(CSI_ENDPOINT)"
            - "--metricsport=8680"
            - "--metricspath=/metrics"
            - "--polltime=60s"
            - "--timeout=3s"
          env:
            - name: CSI_ENDPOINT
              value: unix:///csi/csi-provisioner.sock

```

下面是部分参数解析，详细参数解析请参考：

rbd：https://github.com/ceph/ceph-csi/blob/devel/docs/deploy-rbd.md

cephfs：https://github.com/ceph/ceph-csi/blob/devel/docs/deploy-cephfs.md



nodeid

node的唯一标识，一般填node ip或node name。



type

driver类型，可选项有rbd/cephfs/liveness，对应rbdType/cephfsType/livenessType三个类型的服务。



controllerserver

为true时，启动ControllerServer与IdentityServer。



nodeserver

为true时，启动NodeServer与IdentityServer。



endpoint

ceph-csi组件暴露的grpc服务socket地址，external-provisioner组件将与该socket地址通信，发出创建、删除存储的请求。默认值为unix://tmp/csi.sock。



v

日志输出等级。



drivername

driver名称，与storageclass对象里的provisioner属性值保持一致，默认值为rbd.csi.ceph.com。根据指定driver名称来决定由哪个driver来负责存储的相关操作。



pidlimit

在cgroups中配置PID限制，限制在大量创建、删除存储操作时导致产生大量的PID。-1代表配置限制为最大值，0代表不限制，默认值为0。



enablegrpcmetrics

\[DEPRECATED\]设置为true时，开启grpc metrics。默认值为false。



metricsport

liveness/grpc metrics暴露端口。默认值8080。



metricspath

liveness/grpc metrics暴露url。默认值/metrics。



polltime

存活探测（probe请求）的时间间隔。



timeout

存活探测（probe请求）超时时间。



