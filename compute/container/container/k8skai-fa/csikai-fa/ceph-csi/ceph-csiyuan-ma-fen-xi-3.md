## external-provisioner源码分析（3）-组件启动参数分析

关联链接

external-provisioner组件的源码分析分为三部分：

（1）主体处理逻辑分析；

（2）main方法与Leader选举分析；

（3）组件启动参数分析。



ceph-csi套件分析目录导航



external-provisioner组件启动参数列表

具体参考https://github.com/kubernetes-csi/external-provisioner\#command-line-options

    var (
    	master               = flag.String("master", "", "Master URL to build a client config from. Either this or kubeconfig needs to be set if the provisioner is being run out of cluster.")
    	kubeconfig           = flag.String("kubeconfig", "", "Absolute path to the kubeconfig file. Either this or master needs to be set if the provisioner is being run out of cluster.")
    	csiEndpoint          = flag.String("csi-address", "/run/csi/socket", "The gRPC endpoint for Target CSI Volume.")
    	_                    = deprecatedflags.Add("connection-timeout")
    	volumeNamePrefix     = flag.String("volume-name-prefix", "pvc", "Prefix to apply to the name of a created volume.")
    	volumeNameUUIDLength = flag.Int("volume-name-uuid-length", -1, "Truncates generated UUID of a created volume to this length. Defaults behavior is to NOT truncate.")
    	showVersion          = flag.Bool("version", false, "Show version.")
    	retryIntervalStart   = flag.Duration("retry-interval-start", time.Second, "Initial retry interval of failed provisioning or deletion. It doubles with each failure, up to retry-interval-max.")
    	retryIntervalMax     = flag.Duration("retry-interval-max", 5*time.Minute, "Maximum retry interval of failed provisioning or deletion.")
    	workerThreads        = flag.Uint("worker-threads", 100, "Number of provisioner worker threads, in other words nr. of simultaneous CSI calls.")
    	finalizerThreads     = flag.Uint("cloning-protection-threads", 1, "Number of simultaniously running threads, handling cloning finalizer removal")
    	operationTimeout     = flag.Duration("timeout", 10*time.Second, "Timeout for waiting for creation or deletion of a volume")
    	_                    = deprecatedflags.Add("provisioner")

    	enableLeaderElection    = flag.Bool("enable-leader-election", false, "Enables leader election. If leader election is enabled, additional RBAC rules are required. Please refer to the Kubernetes CSI documentation for instructions on setting up these RBAC rules.")
    	leaderElectionType      = flag.String("leader-election-type", "endpoints", "the type of leader election, options are 'endpoints' (default) or 'leases' (strongly recommended). The 'endpoints' option is deprecated in favor of 'leases'.")
    	leaderElectionNamespace = flag.String("leader-election-namespace", "", "Namespace where the leader election resource lives. Defaults to the pod namespace if not set.")
    	strictTopology          = flag.Bool("strict-topology", false, "Passes only selected node topology to CreateVolume Request, unlike default behavior of passing aggregated cluster topologies that match with topology keys of the selected node.")
    	extraCreateMetadata     = flag.Bool("extra-create-metadata", false, "If set, add pv/pvc metadata to plugin create requests as parameters.")

    	metricsAddress = flag.String("metrics-address", "", "The TCP network address where the prometheus metrics endpoint will listen (example: `:8080`). The default is empty string, which means metrics endpoint is disabled.")
    	metricsPath    = flag.String("metrics-path", "/metrics", "The HTTP path where prometheus metrics will be exposed. Default is `/metrics`.")

    	featureGates        map[string]bool
    	provisionController *controller.ProvisionController
    	version             = "unknown"

    )

    func main() {
    	var config *rest.Config
    	var err error

    	flag.Var(utilflag.NewMapStringBool(&featureGates), "feature-gates", "A set of key=value pairs that describe feature gates for alpha/experimental features. "+
    		"Options are:\n"+strings.Join(utilfeature.DefaultFeatureGate.KnownFeatures(), "\n"))

    	klog.InitFlags(nil)
    	flag.CommandLine.AddGoFlagSet(goflag.CommandLine)
    	flag.Set("logtostderr", "true")
    	flag.Parse()

    ......


#### external-provisioner容器部署的启动参数配置

```
    ...
    args:
        - "--csi-address=$(ADDRESS)"
        - "--v=5"
        - "--timeout=150s"
        - "--retry-interval-start=500ms"
        - "--enable-leader-election=true"
        - "--leader-election-type=leases"
        - "--feature-gates=Topology=true"
    env:
        - name: ADDRESS
            value: unix:///csi/csi-provisioner.sock
    imagePullPolicy: "Always"
    volumeMounts:
    - name: socket-dir
      mountPath: /csi
  volumes：
  - name: socket-dir
    emptyDir: {
      medium: "Memory"
    }
  ...

```

下面是部分参数解析。其他参数请参考：https://github.com/kubernetes-csi/external-provisioner\#command-line-options



csi-address

ceph-csi组件暴露的grpc服务socket地址，external-provisioner组件将与该socket地址通信，发出创建、删除存储的请求。默认值为/run/csi/socket。



timeout

创建存储与删除存储请求的超时时间，默认10秒。



enable-leader-election

是否开启leader选举，配置值为true时开启，默认值为false。



leader-election-type

leader选举时使用的锁类型，包括两种，默认的endpoint与官方推荐的lease。



leader-election-namespace

leader选举所使用的锁对象存储在哪个命令空间，默认存储在external-provisioner pod所在的命名空间。



extra-create-metadata

设置为true后，创建存储时，将额外增加请求参数：pvc的名称、命名空间、pv的名称，默认值为false。



worker-threads

同时执行CreateVolume/DeleteVolume操作的worker数量，默认值为100。



retry-interval-start

创建、删除存储失败后的初始重试时间间隔，默认值为1秒。每一次失败，重试时间间隔会加倍，直到达到--retry-interval-max配置的上限值。



retry-interval-max

创建、删除存储失败后的最大重试时间间隔，默认值为5分钟。



volume-name-prefix

创建pv时给pv名称加的前缀，默认值为pvc。



feature-gates

特性配置。



配置示例：--feature-gates=Topology=true，存储拓扑相关，具体使用请参考：

https://kubernetes.io/zh/blog/2018/10/11/kubernetes-%E4%B8%AD%E7%9A%84%E6%8B%93%E6%89%91%E6%84%9F%E7%9F%A5%E6%95%B0%E6%8D%AE%E5%8D%B7%E4%BE%9B%E5%BA%94/

https://kubernetes-csi.github.io/docs/topology.html



kube-api-qps与kube-api-burst

用于kube-client与kube-apiserver通信时客户端kube-client的限流。

限流实现上是令牌桶算法，kube-api-qps可以看作是每秒产生令牌的速度，而kube-api-burst可以看作是令牌桶的大小，kube-client发送给kube-apiserver的请求需要拿到令牌桶里的令牌后才能发送。

其中kube-api-qps默认值为5，kube-api-burst默认值为10。



