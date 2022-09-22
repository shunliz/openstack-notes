# Helm

作为CNCF的毕业项目。它的官方的定义是：Helm是一个为K8s进行包管理的工具。Helm将yaml作为一个整体管理并实现了这些yaml的高效复用，就像Linux中的yum或apt-get，它使我们能够在K8s中方便快捷的安装、管理、卸载K8s应用。

Helm基于go模板语言，用户只要提供规定的目录结构和模板文件。在真正部署时Helm模板引擎便可以将其渲染成真正的K8s资源配置文件，并按照正确的顺序将它们部署到节点上。

Helm中有三个重要概念，分别为Chart、Repository和Release。

Chart代表中Helm包。它包含在K8s集群内部运行应用程序，工具或服务所需的所有资源定义。可以类比成yum中的RPM。

Repository就是用来存放和共享Chart的地方，可以类比成Maven仓库。

Release是运行在K8s集群中的Chart的实例，一个Chart可以在同一个集群中安装多次。Chart就像流水线中初始化好的模板，Release就是这个“模板”所生产出来的各个产品。

Helm作为K8s的包管理软件，每次安装Charts 到K8s集群时，都会创建一个新的 release。你可以在Helm 的Repository中寻找需要的Chart。Helm对于部署过程的优化的点在于简化了原先完成配置文件编写后还需使用一串kubectl命令进行的操作、统一管理了部署时的可配置项以及方便了部署完成后的升级和维护。

# **Helm的架构**

![](/assets/compute-container-helmtiller1.png)

Helm客户端使用REST+JSON的方式与K8s中的apiserver进行交互，进而管理deployment、service等资源，并且客户端本身并不需要数据库，它会把相关的信息储存在K8s集群内的Secrets中。



假设我们的Chart名称叫做myChart，我们可以使用命令：

```
$ Helm create myChart
```

创建一个初始模板工程，那么在名为myChart的目录下包含了以下目录和文件：

其中关键的目录和文件作用如下：

★ templates/ 目录包含了模板文件。Helm会通过模板渲染引擎渲染所有该目录下的文件来生成Chart，之后将收集到的模板渲染结果发送给K8s。

★ values.yaml 文件对于模板也非常重要。这个文件包含了对于一个Chart的默认值 。这些值可以在用户执行Helm install 或 Helm upgrade时指定新的值来进行覆盖。

★ Chart.yaml 文件包含对于该Chart元数据描述。这些描述信息可以在模板中被引用。

★ \_helper.tpl 包含了一些可以在Chart中进行复用的模板定义。

★ 其他诸如deployment.yaml、service.yaml、ingress.yaml文件，就是我们用于生成K8s配置文件的模板，Helm默认会按照如下的顺序将生成资源配置发送给K8s:



```
Namespace -> NetworkPolicy -> ResourceQuota -> LimitRange -> PodSecurityPolicy --> PodDisruptionBudget -> 
ServiceAccount -> Secret -> SecretList -> ConfigMap -> StorageClass -> PersistentVolume -> PersistentVolumeClaim ->
 CustomResourceDefinition -> ClusterRole -> ClusterRoleList -> ClusterRoleBinding -> ClusterRoleBindingList -> 
 Role -> RoleList -> RoleBinding -> RoleBindingList -> Service -> DaemonSet -> Pod -> ReplicationController -> 
 ReplicaSet -> Deployment -> HorizontalPodAutoscaler -> StatefulSet -> Job -> CronJob -> Ingress -> 
 APIService
```



