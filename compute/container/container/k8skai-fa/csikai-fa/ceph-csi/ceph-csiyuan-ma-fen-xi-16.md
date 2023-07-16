# kubelet分析-csi driver注册源码分析 {#articleContentId}

kubeletcsi driver注册分析

kubelet注册csi driver的相关功能代码与kubelet的pluginManager有关，所以接下来对pluginManager进行分析。分析将分为pluginManager的初始化分析以及pluginManager的运行（处理逻辑）分析。



基于tag v1.17.4

https://github.com/kubernetes/kubernetes/releases/tag/v1.17.4



kubelet注册csi driver的原理

kubelet的pluginManager会监听某个特定目录，而负责向kubelet注册csi driver的组件Node Driver Registrar会创建暴露服务的socket在该目录下（每个plugin会对应一个Node Driver Registrar组件，也就是说，一个Node Driver Registrar只负责一个plugin的注册工作），pluginManager通过Node Driver Registrar组件暴露的socket获取plugin信息（包括plugin的socket地址、plugin名称等），从而最终做到根据该目录下socket文件的新增/删除来做相应的plugin注册/取消注册操作。

![](/assets/compute-container-k8s-cephcsi12622161.png)该图是Node Driver Registrar向kubelet注册csi driver的步骤流程图，这里大概看一下，具体请看这篇博客：Node Driver Registrar源码分析，结合本篇博客一起理解kubelet注册csi driver的整个过程。



plugin注册完成后，后续kubelet将通过plugin暴露的socket与plugin进行通信，做存储挂载/解除挂载等操作。



**Node Driver Registrar**

Node Driver Registrar在前面的文章中介绍过，它是一个sidecar容器，通过Kubelet的插件注册机制将CSI plugin（csi driver，两个名词意义一样）注册到Kubelet，让kubelet做volume的mount/umount操作时知道怎么调用相应的csi plugin。



**kubelet pluginManager源码分析**

**1 pluginManager的初始化**

调用NewMainKubelet\(\)初始化kubelet的时候，会调用pluginmanager.NewPluginManager来初始化pluginManager，所以把NewMainKubelet\(\)作为分析入口。



NewMainKubelet\(\)

NewMainKubelet\(\)中调用了pluginmanager.NewPluginManager来初始化pluginManager。



这里留意klet.getPluginsRegistrationDir\(\)，调用该方法实际会返回plugins\_registry，而该sockDir会传参进入pluginManager的desiredStateOfWorldPopulator结构体当中，相当于pluginManager会监听plugins\_registry目录（负责向kubelet注册csi driver的组件Node Driver Registrar会创建暴露服务的socket在该目录下），pluginManager通过Node Driver Registrar组件暴露的socket获取plugin信息（包括plugin的socket地址、plugin名称等），从而最终做到根据该目录下socket文件的新增/删除来做相应的plugin注册/取消注册操作。

```
// pkg/kubelet/kubelet.go
// NewMainKubelet instantiates a new Kubelet object along with all the required internal modules.
// No initialization of Kubelet and its modules should happen here.
func NewMainKubelet(kubeCfg *kubeletconfiginternal.KubeletConfiguration,
	...
	nodeStatusMaxImages int32) (*Kubelet, error) {
	...
    klet.pluginManager = pluginmanager.NewPluginManager(
		klet.getPluginsRegistrationDir(), /* sockDir */
		kubeDeps.Recorder,
	)
	...
}

```

```
// pkg/kubelet/pluginmanager/plugin_manager.go
// NewPluginManager returns a new concrete instance implementing the
// PluginManager interface.
func NewPluginManager(
	sockDir string,
	recorder record.EventRecorder) PluginManager {
	asw := cache.NewActualStateOfWorld()
	dsw := cache.NewDesiredStateOfWorld()
	reconciler := reconciler.NewReconciler(
		operationexecutor.NewOperationExecutor(
			operationexecutor.NewOperationGenerator(
				recorder,
			),
		),
		loopSleepDuration,
		dsw,
		asw,
	)

	pm := &pluginManager{
		desiredStateOfWorldPopulator: pluginwatcher.NewWatcher(
			sockDir,
			dsw,
		),
		reconciler:          reconciler,
		desiredStateOfWorld: dsw,
		actualStateOfWorld:  asw,
	}
	return pm
}

```

###### klet.getPluginsRegistrationDir\(\)

调用klet.getPluginsRegistrationDir\(\)会返回`plugins_registry`。

```
// pkg/kubelet/kubelet_getters.go
// getPluginsRegistrationDir returns the full path to the directory under which
// plugins socket should be placed to be registered.
// More information is available about plugin registration in the pluginwatcher
// module
func (kl *Kubelet) getPluginsRegistrationDir() string {
	return filepath.Join(kl.getRootDir(), config.DefaultKubeletPluginsRegistrationDirName)
}

```

```
// pkg/kubelet/config/defaults.go
const (
    ...
    DefaultKubeletPluginsRegistrationDirName = "plugins_registry"
    ...
)

```

**2 pluginManager struct**

再来看到pluginManager结构体，pluginManager结构体与volumeManager结构体类似，都有actualStateOfWorld与desiredStateOfWorld两个属性。



kubelet pluginManager监听的socket注册目录每增加/删除一个socket文件，都会写入desiredStateOfWorld中/从desiredStateOfWorld中删除。

#### actualStateOfWorld

actualStateOfWorld结构体中存放的是已经完成了plugin注册操作的Node Driver Registrar组件暴露的socket相关信息。

#### desiredStateOfWorld

desiredStateOfWorld结构体中存放的是在pluginManager监听目录下存在的，希望完成plugin注册操作的Node Driver Registrar组件暴露的socket相关信息。



**3 pluginManager的运行**

上面介绍了pluginManager的初始化，接下来介绍pluginManager的运行也即Run方法进行分析，分析一下pluginManager的处理逻辑。



因为调用逻辑比较复杂，这里直接跳过了调用过程的分析，直接进入kl.pluginManager.Run\(\)的分析，下面只给出该方法的一个调用链：

kubelet的Run\(\)方法（pkg/kubelet/kubelet.go） --&gt; kl.updateRuntimeUp\(\)（pkg/kubelet/kubelet.go） --&gt; kl.initializeRuntimeDependentModules\(\)（pkg/kubelet/kubelet.go） --&gt; kl.pluginManager.Run\(\)



**kl.pluginManager.Run**

下面直接看到kl.pluginManager.Run的代码。

该方法主要逻辑有两个：

（1）pm.desiredStateOfWorldPopulator.Start\(\)：持续监听plugin的socket注册目录的变化事件，将Node Driver Registrar的socket信息写入desiredStateOfWorld中/从desiredStateOfWorld中删除；

（2）pm.reconciler.Run\(\)。

```
// pkg/kubelet/pluginmanager/plugin_manager.go
func (pm *pluginManager) Run(sourcesReady config.SourcesReady, stopCh <-chan struct{}) {
	defer runtime.HandleCrash()

	pm.desiredStateOfWorldPopulator.Start(stopCh)
	klog.V(2).Infof("The desired_state_of_world populator (plugin watcher) starts")

	klog.Infof("Starting Kubelet Plugin Manager")
	go pm.reconciler.Run(stopCh)

	metrics.Register(pm.actualStateOfWorld, pm.desiredStateOfWorld)
	<-stopCh
	klog.Infof("Shutting down Kubelet Plugin Manager")
}

```

**3.1 pm.desiredStateOfWorldPopulator.Start\(\)**

跑一个goroutine，持续监听plugin的socket注册目录的变化事件：

（1）当变化事件为新增事件时，即socket目录下多了文件，则调用w.handleCreateEvent，将该socket加入到desiredStateOfWorld中；

（2）当变化事件为删除事件时，即socket目录下删除了文件，则调用w.handleDeleteEvent，将该socket从desiredStateOfWorld中删除。



**w.handleCreateEvent\(\)**

w.handleCreateEvent\(\)主要逻辑：

（1）判断新增事件是否为文件，且是否是socket文件；

（2）是socket文件，则调用w.handlePluginRegistration做处理，主要是将该socket加入到desiredStateOfWorld中。

###### w.handleDeleteEvent\(\)

w.handleDeleteEvent\(\)主要逻辑：  
（1）将socket从desiredStateOfWorld中删除。



**3.2 pm.reconciler.Run\(\)**

pm.reconciler.Run\(\)主要逻辑为对比desiredStateOfWorld与actualStateOfWorld做调谐，做plugin的注册操作/取消注册操作。具体逻辑如下：

（1）对比actualStateOfWorld，如果desiredStateOfWorld中没有该socket信息，或者desiredStateOfWorld中该socket的Timestamp值与actualStateOfWorld中的不相等（即plugin更新了），则说明该plugin需要取消注册（更新的plugin需先取消注册，然后再次注册），调用rc.operationExecutor.UnregisterPlugin做plugin取消注册操作；

（2）对比desiredStateOfWorld，如果actualStateOfWorld中没有该socket信息，则调用rc.operationExecutor.RegisterPlugin做plugin注册操作。



**3.2.1 rc.operationExecutor.UnregisterPlugin\(\)**

rc.operationExecutor.UnregisterPlugin\(\)主要逻辑：做plugin取消注册操作。



那plugin取消注册操作具体做了什么呢？继续往下分析。



**plugin取消注册操作方法调用链**

kl.pluginManager.Run --&gt; pm.desiredStateOfWorldPopulator.Start\(\) --&gt; pm.reconciler.Run\(\) --&gt; rc.reconcile\(\) --&gt; rc.operationExecutor.UnregisterPlugin\(\) --&gt; oe.operationGenerator.GenerateUnregisterPluginFunc\(\) --&gt; handler.DeRegisterPlugin\(\) --&gt; nim.UninstallCSIDriver\(\) --&gt; nim.updateNode\(\)



下面来对plugin取消注册操作的部分关键方法进行分析。



**GenerateUnregisterPluginFunc**

下面来分析下GenerateUnregisterPluginFunc的逻辑，主要是定义并实现一个plugin取消注册的方法，然后返回。plugin取消注册方法主要逻辑如下：

（1）检测Node Driver Registrar组件socket的连通性；

（2）通过Node Driver Registrar组件socket获取plugin信息；

（3）从actualStateOfWorld中删除该Node Driver Registrar组件的socket信息；

（4）调用handler.DeRegisterPlugin做进一步的plugin取消注册操作。



所以接下来会对handler.DeRegisterPlugin方法进行分析。

**handler.DeRegisterPlugin\(\)**

handler.DeRegisterPlugin\(\)方法里逻辑比较简单，主要是调用了unregisterDriver\(\)方法。



unregisterDriver\(\)方法主要逻辑：

（1）从csiDrivers变量中删除该plugin信息（后续kubelet调用csi plugin进行存储的挂载/解除挂载操作，将通过plugin名称从csiDrivers变量中拿到socket地址并进行通信，所以取消注册plugin时，需要从csiDrivers变量中把该plugin信息去除）；

（2）调用nim.UninstallCSIDriver\(\)做进一步处理。

**nim.UninstallCSIDriver\(\)**

接下来看到nim.UninstallCSIDriver\(\)方法的分析。



nim.UninstallCSIDriver\(\)中主要看到nim.uninstallDriverFromCSINode\(\)、removeMaxAttachLimit\(\)与removeNodeIDFromNode\(\)3个方法，主要逻辑都在其中：

（1）nim.uninstallDriverFromCSINode\(\)：更新CSINode对象，从中去除取消注册的plugin的相关信息。

（2）removeMaxAttachLimit\(\)：更新node对象，从node.Status.Capacity及node.Status.Allocatable中去除取消注册的plugin的相关信息。

（3）removeNodeIDFromNode\(\)：更新node对象，从node对象的annotation中key为csi.volume.kubernetes.io/nodeid的值中去除取消注册的plugin信息。

CSIDriver对象示例：

```
apiVersion: storage.k8s.io/v1
kind: CSINode
metadata:
  name: 192.168.1.10
spec:
  drivers:
  - name: cephfs.csi.ceph.com
    nodeID: 192.168.1.10
    topologyKeys: null
  - name: rbd.csi.ceph.com
    nodeID: 192.168.1.10
    topologyKeys: null

```

nim.UninstallCSIDriver\(\)源码：

（1）nim.uninstallDriverFromCSINode\(\)：更新CSINode对象，从中去除取消注册的plugin的相关信息。

（2）removeMaxAttachLimit\(\)：更新node对象，从node.Status.Capacity及node.Status.Allocatable中去除取消注册的plugin的相关信息。

（3）removeNodeIDFromNode\(\)：更新node对象，从node对象的annotation中key为`csi.volume.kubernetes.io/nodeid`

的值中去除取消注册的plugin信息。



**3.2.2 rc.operationExecutor.RegisterPlugin\(\)**

rc.operationExecutor.RegisterPlugin\(\)主要逻辑：做plugin注册操作。



那plugin注册操作具体做了什么呢？继续往下分析。



**plugin注册操作方法调用链**

kl.pluginManager.Run --&gt; pm.desiredStateOfWorldPopulator.Start\(\) --&gt; pm.reconciler.Run\(\) --&gt; rc.reconcile\(\) --&gt; rc.operationExecutor.RegisterPlugin\(\) --&gt; oe.operationGenerator.GenerateRegisterPluginFunc\(\) --&gt; handler.RegisterPlugin\(\) --&gt; nim.InstallCSIDriver\(\) --&gt; nim.updateNode\(\)



下面来对plugin注册操作的部分关键方法进行分析。



**GenerateRegisterPluginFunc**

下面来分析下GenerateRegisterPluginFunc的逻辑，主要是定义并实现一个plugin注册的方法，然后返回。plugin注册方法主要逻辑如下：

（1）检测Node Driver Registrar组件socket的连通性；

（2）通过Node Driver Registrar组件socket获取plugin信息（包括plugin的socket地址、plugin名称等）；

（3）调用handler.ValidatePlugin\(\)，检查已注册的plugin中是否有比该需要注册的plugin同名的的更高的版本，如有，则返回注册失败，并通知plugin注册失败；

（4）向actualStateOfWorld中增加该Node Driver Registrar组件的socket信息；

（5）调用handler.RegisterPlugin\(\)做进一步的plugin注册操作;

（6）调用og.notifyPlugin，通知plugin，已经向kubelet注册成功/注册失败。



所以接下来会对handler.RegisterPlugin\(\)方法进行分析。



**handler.RegisterPlugin\(\)**

handler.DeRegisterPlugin\(\)方法主要逻辑：

（1）存储该plugin信息（主要是plugin名称与plugin的socket地址）到csiDrivers变量中（后续kubelet调用csi plugin进行存储的挂载/解除挂载操作，将通过plugin名称从此变量中拿到socket地址并进行通信）；

（2）检测Node Driver Registrar组件socket的连通性；

（3）通过plugin的socket获取plugin信息（包括plugin的NodeId、最大挂载数量限制、拓扑信息等）；

（4）调用nim.InstallCSIDriver，做进一步的plugin注册操作。

**nim.InstallCSIDriver\(\)**

nim.InstallCSIDriver\(\)中主要看到updateNodeIDInNode\(\)与nim.updateCSINode\(\)两个方法，主要逻辑都在其中：

（1）updateNodeIDInNode\(\)：更新node对象，向node对象的annotation中key为csi.volume.kubernetes.io/nodeid的值中去增加注册的plugin信息。

（2）nim.updateCSINode\(\)：创建或更新CSINode对象。



**总结**

本节主要讲解了kubelet注册csi driver的原理，以及其代码的分析，也顺带提了一下Node Driver Registrar组件，下面来做个总结。



**Node Driver Registrar**

Node Driver Registrar在后面的文章中会介绍，Node Driver Registrar源码分析，它是一个sidecar容器，通过Kubelet的插件注册机制将CSI plugin（csi driver，两个名词意义一样）注册到Kubelet，让kubelet做volume的mount/umount操作时知道怎么调用相应的csi plugin。



**kubelet注册csi driver的原理**

kubelet的pluginManager会监听某个特定目录，而负责向kubelet注册csi driver的组件Node Driver Registrar会创建暴露服务的socket在该目录下（每个plugin会对应一个Node Driver Registrar组件，也就是说，一个Node Driver Registrar只负责一个plugin的注册工作），pluginManager通过Node Driver Registrar组件暴露的socket获取plugin信息（包括plugin的socket地址、plugin名称等），从而最终做到根据该目录下socket文件的新增/删除来做相应的plugin注册/取消注册操作。



plugin注册完成后，后续kubelet将通过plugin暴露的socket与plugin进行通信，做存储挂载/解除挂载等操作。



下面再来总结一下在kubelet的pluginManager中，plugin的注册/取消注册操作分别做了什么动作。



**plugin注册操作**

（1）存储该plugin信息（主要是plugin名称与plugin的socket地址）到csiDrivers变量中（后续kubelet调用csi plugin进行存储的挂载/解除挂载操作，将通过plugin名称从此变量中拿到socket地址并进行通信；csiDriver变量代码位置-pkg/volume/csi/csi\_plugin.go）；

（2）更新node对象，向node对象的annotation中key为csi.volume.kubernetes.io/nodeid的值中去增加注册的plugin信息。

（3）创建或更新CSINode对象。



**plugin取消注册操作**

（1）从csiDrivers变量中删除该plugin信息（后续kubelet调用csi plugin进行存储的挂载/解除挂载操作，将通过plugin名称从csiDrivers变量（csiDriver变量代码位置-pkg/volume/csi/csi\_plugin.go）中拿到socket地址并进行通信，所以取消注册plugin时，需要从csiDrivers变量中把该plugin信息去除）；

（2）更新CSINode对象，从中去除取消注册的plugin的相关信息。

（3）更新node对象，从node.Status.Capacity及node.Status.Allocatable中去除取消注册的plugin的相关信息。

（4）更新node对象，从node对象的annotation中key为csi.volume.kubernetes.io/nodeid的值中去除取消注册的plugin信息。





