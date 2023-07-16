# kubelet分析-csi driver注册分析-Node Driver Registrar源码分析 {#articleContentId}

**Node Driver Registrar分析**

node-driver-registrar是一个sidecar容器，通过Kubelet的插件注册机制将CSI plugin（csi driver，两个名词意义一样）注册到Kubelet，让kubelet做volume的mount/umount操作时知道怎么调用相应的csi plugin。

Node Driver Registrar的内容相对简单，将在本文中对其作用、源码、组件间调用逻辑等进行分析。

repo：[https://github.com/kubernetes-csi/node-driver-registrar](https://github.com/kubernetes-csi/node-driver-registrar)

**Node Driver Registrar启动参数  
**

```
// cmd/csi-node-driver-registrar/main.go
// Command line flags
var (
    connectionTimeout       = flag.Duration("connection-timeout", 0, "The --connection-timeout flag is deprecated")
    csiAddress              = flag.String("csi-address", "/run/csi/socket", "Path of the CSI driver socket that the node-driver-registrar will connect to.")
    kubeletRegistrationPath = flag.String("kubelet-registration-path", "", "Path of the CSI driver socket on the Kubernetes host machine.")
    showVersion             = flag.Bool("version", false, "Show version.")
    version                 = "unknown"

    // List of supported versions
    supportedVersions = []string{"1.0.0"}
)
```

下面讲解下最关键的两个启动参数，主要是配置2个socket地址。

（1）csi-address

csi plugin组件暴露的grpc服务socket地址。

（2）kubelet-registration-path

Node Driver Registrar容器暴露的grpc服务socket地址，同时kubelet也会通过该socket地址访问Node Driver Registrar容器的接口，主要用于向kubelet注册csi plugin。

Node Driver Registrar容器权限

（1）RBAC：Node Driver Registrar容器没有访问kubernetes API的需求，所以不用做相关的RBAC配置。

（2）需要将前面提到的2个socket的父目录作为hostPath挂载进容器中，并对socket拥有创建、删除、访问等权限（通过配置containers\[\].securityContext.privileged=true获得权限）。

Node Driver Registrar容器部署yaml

Node-Driver-Registrar容器与csi plugin NodeServer容器一起，使用daemonset部署，即每个node节点都有。

Node-Driver-Registrar容器的部署yaml如下：

```
    containers:
        - name: driver-registrar-rbd
          # This is necessary only for systems with SELinux, where
          # non-privileged sidecar containers cannot access unix domain socket
          # created by privileged CSI driver container.
          securityContext:
            privileged: true
          image: quay.io/k8scsi/csi/csi-node-driver-registrar:v1.3.0
          args:
            - "--v=5"
            - "--csi-address=/csi/csi.sock"
            - "--kubelet-registration-path=/var/lib/kubelet/plugins/rbd.csi.ceph.com/csi.sock"
          lifecycle:
            preStop:
              exec:
                command: [
                  "/bin/sh", "-c",
                  "rm -rf /registration/rbd.csi.ceph.com \
                  /registration/rbd.csi.ceph.com-reg.sock"
                ]
          env:
            - name: KUBE_NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
          volumeMounts:
            - name: socket-dir
              mountPath: /csi
            - name: registration-dir
              mountPath: /registration
          imagePullPolicy: "Always"
    volumes:
        - name: socket-dir
          hostPath:
            path: /var/lib/kubelet/plugins/rbd.csi.ceph.com
            type: DirectoryOrCreate
        - name: registration-dir
          hostPath:
            path: /var/lib/kubelet/plugins_registry/
            type: Directory
```

## Node Driver Registrar注册csi plugin driver步骤

![](/assets/compute-container-k8s-cephcsi12622161.png)第 1 步

Node Driver Registrar连接csi plugin组件暴露的grpc服务socket地址，调用GetPluginInfo接口，获取csi plugin的driver名称。



第 2 步

在 kubelet-registration-path目录下启动一个 socket，对外暴露GetInfo 和 NotifyRegistrationStatus 两个接口。kubelet 通过 Watcher可以发现该socket。



第 3 步

kubelet 通过 Watcher监控/var/lib/kubelet/plugins\_registry/目录，发现上述socket 后，通过该 socket 调用 Node-Driver-Registrar 的 GetInfo 接口，获取csi plugin组件暴露的grpc服务socket地址以及csi plugin组件的driver名称。



第 4 步

kubelet 通过csi plugin组件暴露的grpc服务socket地址对其NodeGetInfo接口进行调用，获取csi plugin的nodeID等信息。



第 5 步

kubelet根据上一步获得的信息，去更新node节点的 Annotations、Labels、status.allocatable 等信息，同时创建（或更新）一个 CSINode 对象。



第 6 步

kubelet通过socket调用Node-Driver-Registrar容器的NotifyRegistrationStatus接口，通知注册csi plugin成功。



通过以上 6 步就实现了 csi plugin注册机制。



**Node Driver Registrar源码分析**

下面结合Node Driver Registrar的源码对注册csi plugin driver步骤进行详细分析。主要分析main\(\)、nodeRegister\(\)、GetInfo\(\)与NotifyRegistrationStatus\(\)。



**1.main\(\)**

先从main\(\)入手。



主要逻辑：

（1）组件启动参数校验；

（2）连接csi plugin组件暴露的grpc服务socket地址，调用GetPluginInfo接口，获取csi plugin的driver名称；

（3）调用nodeRegister方法做csi plugin注册操作。

```
// cmd/csi-node-driver-registrar/main.go
func main() {
	klog.InitFlags(nil)
	flag.Set("logtostderr", "true")
	flag.Parse()

	if *kubeletRegistrationPath == "" {
		klog.Error("kubelet-registration-path is a required parameter")
		os.Exit(1)
	}

	if *showVersion {
		fmt.Println(os.Args[0], version)
		return
	}
	klog.Infof("Version: %s", version)

	if *connectionTimeout != 0 {
		klog.Warning("--connection-timeout is deprecated and will have no effect")
	}

	// Once https://github.com/container-storage-interface/spec/issues/159 is
	// resolved, if plugin does not support PUBLISH_UNPUBLISH_VOLUME, then we
	// can skip adding mapping to "csi.volume.kubernetes.io/nodeid" annotation.

	klog.V(1).Infof("Attempting to open a gRPC connection with: %q", *csiAddress)
	csiConn, err := connection.Connect(*csiAddress)
	if err != nil {
		klog.Errorf("error connecting to CSI driver: %v", err)
		os.Exit(1)
	}

	klog.V(1).Infof("Calling CSI driver to discover driver name")
	ctx, cancel := context.WithTimeout(context.Background(), csiTimeout)
	defer cancel()

	csiDriverName, err := csirpc.GetDriverName(ctx, csiConn)
	if err != nil {
		klog.Errorf("error retreiving CSI driver name: %v", err)
		os.Exit(1)
	}

	klog.V(2).Infof("CSI driver name: %q", csiDriverName)

	// Run forever
	nodeRegister(csiDriverName)
}

```

**2.nodeRegister\(\)**

nodeRegister\(\)用于向kubelet注册csi plugin。主要逻辑：

（1）调用newRegistrationServer\(\)初始化registrationServer结构体；

（2）在 kubelet-registration-path目录下启动一个 socket，对外暴露GetInfo 和 NotifyRegistrationStatus 两个接口（kubelet 通过 Watcher可以发现该socket）。

#### 3.GetInfo\(\)

GetInfo\(\)：kubelet 通过调用 Node-Driver-Registrar 的 GetInfo 接口，获取csi plugin组件暴露的grpc服务socket地址以及csi plugin组件的driver名称。

#### 4.NotifyRegistrationStatus\(\)

NotifyRegistrationStatus\(\)：kubelet通过调用Node-Driver-Registrar的NotifyRegistrationStatus接口，通知注册csi plugin成功。





**总结**

**Node Driver Registrar作用**

node-driver-registrar是一个sidecar容器，通过Kubelet的插件注册机制将CSI plugin（csi driver，两个名词意义一样）注册到Kubelet，让kubelet做volume的mount/umount操作时知道怎么调用相应的csi plugin。



**Node Driver Registrar涉及的两个socket**

**（1）csi-address**

csi plugin组件暴露的grpc服务socket地址。



**（2）kubelet-registration-path**

Node Driver Registrar容器暴露的grpc服务socket地址，同时kubelet也会通过该socket地址访问Node Driver Registrar容器的接口，主要用于向kubelet注册csi plugin。



**关于cluster-driver-registrar**

旧的注册组件driver-registrar已被cluster-driver-registrar与node-driver-registrar两个组件替代，其中cluster-driver-registrar也已经deprecated。



注册组件可分为cluster-driver-registrar与node-driver-registrar，两者的作用不同，自动/手工创建CSIDriver object替代了cluster-driver-registrar组件的作用，node-driver-registrar任然需要，具体作用可以参考下方组件的readme链接。



driver-registrar：https://github.com/kubernetes-csi/driver-registrar\#readme

cluster-driver-registrar：https://github.com/kubernetes-csi/cluster-driver-registrar\#readme

node-driver-registrar：https://github.com/kubernetes-csi/node-driver-registrar\#readme



