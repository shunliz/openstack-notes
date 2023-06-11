## Operator Framework

Operator Framework 是 CoreOS 开源的一个用于快速开发维护 Operator 的工具包，该框架包含三个主要的项目：

* Operator SDK: 无需了解复杂的 Kubernetes API 特性，快速构建一个 Operator 应用。
* Operator Lifecycle Manager \(OLM\): 帮助安装、更新和管理跨集群的运行中的所有 Operator（以及他们的相关服务）
* Operator Metering: 收集 operator metric 信息，并生成报告

通过这三个子项目，Operator Framework 完成了 operator 的生成、维护、监控全过程。下面我们逐个了解一下 Operator Framework 全家桶分别是如何工作的。

## Operator SDK

Operator-sdk 帮助我们一键生成 operator 的框架，留出部分函数让我们来完成业务逻辑。

### 安装 operator-sdk

在 MacOS 系统上开发的话只需要一行命令即可安装：

```
brew install operator-sdk
```

Linux 系统可以通过二进制源文件安装：

```
$ RELEASE_VERSION=v0.10.0
$ curl -OJL https://github.com/operator-framework/operator-sdk/releases/download/${RELEASE_VERSION}/operator-sdk-${RELEASE_VERSION}-x86_64-linux-gnu
$ chmod +x operator-sdk-${RELEASE_VERSION}-x86_64-linux-gnu && sudo mkdir -p /usr/local/bin/ && sudo cp operator-sdk-${RELEASE_VERSION}-x86_64-linux-gnu /usr/local/bin/operator-sdk && rm operator-sdk-${RELEASE_VERSION}-x86_64-linux-gnu
```

### 初始化项目

Operator 是用 Go 语言开发的，所以在开发项目前，需要安装好 Go 语言环境。而 Operator-sdk 初始化项目只需要几行代码：

```
$ mkdir -p $HOME/projects/
$ cd $HOME/projects/
$ export GO111MODULE=on
$
$ operator-sdk new app-operator --repo github.com/zwwhdls/app-operator --vendor
$ cd app-operator
```

### 生成 CRD 及其 controller

Operator-sdk 主要会帮你生成 CRD 文件和其 controller 框架。

生成 CRD 命令的格式：

```
$ operator-sdk add api --api-version=<自定义资源的 api version> --kind <自定义资源的 kind>
$ operator-sdk add api --api-version=o0w0o.cn/v1 --kind App
```

然后在项目中会发现多了几个文件，其中有一个`.../app-operator/pkg/apis/o0w0o/v1/app_types.go`，其中帮你创建好了 App 的资源定义：

```go
type AppSpec struct {
}

type AppStatus struct {
}

type App struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   AppSpec   `json:"spec,omitempty"`
    Status AppStatus `json:"status,omitempty"`
}
```

而你需要做的是定义`AppSpec`和`AppStatus`，分别是描述资源的属性和状态。定义好了之后只需要执行下面这条命令，即可在 CRD Yaml 中自动生成你定义的字段。

```
$ operator-sdk generate k8s
```

定义好 CR 之后，我们需要告诉 K8s 创建这个资源的时候需要做什么，也就是需要构建 controller。Operator-sdk 可以一键生成 controller 的框架：

```
$ operator-sdk add controller --api-version=o0w0o.cn/v1 --kind=App
INFO[0000] Generating controller version app.o0w0o.cn/v1 for kind App.
INFO[0000] Created pkg/controller/app/app_controller.go
INFO[0000] Created pkg/controller/add_app.go
INFO[0000] Controller generation complete.
```

看日志可以知道，Operator 帮我们创建了两个文件。

其中，`add_app.go`是用来将 controller 注册进 manager 中的，相当于帮你生成了一个 K8s 的控制器。  
而`app_controller.go`则是真正定义 controller 的地方。在这个文件里，有两个标注了`TODO(User)`的地方，顾名思义，只有这两个地方需要用户自己写。

```go
func add(mgr manager.Manager, r reconcile.Reconciler) error {
    ... 
    // TODO(user): Modify this to be the types you create that are owned by the primary resource
    ...
    return nil
}

// TODO(user): Modify this Reconcile function to implement your Controller logic.  This example creates
func (r *ReconcileApp) Reconcile(request reconcile.Request) (reconcile.Result, error) {
    ...
}
```

首先我们要知道，operator 的工作模式是基于事件的，监听到特定的事件后，触发对应的操作。那就不难理解，第一个 TODO 就是定义需要监听哪些资源；第二个 TODO 就是定义监听到这些资源的事件后，需要做哪些操作。

### 部署

Operator 的部署 Yaml 都是 Operator-sdk 帮你生成好的。在完成 Operator 的开发工作后，只需要将`operator.yaml`中的 image 替换掉，再通过如下步骤部署即可。

```
// 创建 RBAC 相关
$ kubectl create -f deploy/service_account.yaml
$ kubectl create -f deploy/role.yaml
$ kubectl create -f deploy/role_binding.yaml

// 部署 operator CRD
$ kubectl apply -f deploy/crds/app_v1_app_crd.yaml

// 部署 operator
$ kubectl create -f deploy/operator.yaml
```

## Operator Lifecycle Manager

OLM 扩展了Kubernetes，提供了一种陈述式的方式来安装，管理和升级Operator，以及它在集群中所依赖的资源。它还对其管理的组件强制执行一些约束，以确保良好的用户体验。

该项目使用户能够执行以下操作：

* 将应用程序定义为封装了需求和元数据的一个 Kubernetes 资源
* 使用依赖项解析方式自动地安装应用程序，或者使用 kubectl/oc\(OpenShift Client\) 手动安装应用程序
* 使用不同的批准策略自动升级应用程序

### 定义的 CR

| **Resource** | **Short name** | **Owner** | **Description** |
| :--- | :--- | :--- | :--- |
| ClusterServiceVersion | `csv` | OLM | OLM 管理的 operator 的基本信息等，包括版本信息、其管理的 CRD、必须安装的 CRD 、依赖、安装方式等 |
| InstallPlan | `ip` | Catalog | 计算要创建的资源列表，以便自动安装或升级CSV |
| CatalogSource | `catsrc` | Catalog | 定义应用程序的CSV，CRD和 package 的存储库。 |
| Subscription | `sub` | Catalog | 记录何时及如何更新 CSV 等信息，相当于订阅需要观察的 CSV |
| OperatorGroup | `og` | OLM | 配置与 OperatorGroup 对象在同一名称空间中部署的所有 Operator，以在名称空间列表或群集范围内查看其 CR |

ClusterServiceVersion 可以被收集到 CatalogSource，它通过 InstallPlan 解析依赖实现安装的自动化，并且通过 Subscription 保持使用最新版本。

### 定义的 operator

OLM 定义了两个 operator 来维护自定义 operator 的生命周期，分别为：

* OLM operator：创建 CSV 中定义的应用，监听 CSV 的变化，并维持其定义的状态。

* Catalog Operator：解析并安装 CSV 及其依赖的资源；监听 channels 中 CatalogSources 的变化，并将其更新到最新可用版本；

### 使用过程

在你的 operator 项目中生成 CSV 和 package 的 yaml：

```
operator-sdk olm-catalog gen-csv --csv-version xx.yy.zz
```

需要同时生成 CRD 的话，在上面的命令后面加上：

```
--update-crds
```

在本地测试 operator 是否合法：﻿[https://github.com/operator-framework/community-operators/blob/master/docs/testing-operators.md\#manual﻿-testing-on-kubernetes﻿﻿](https://github.com/operator-framework/community-operators/blob/master/docs/testing-operators.md#manual﻿-testing-on-kubernetes﻿﻿)

最后在 GitHub 上 upstream-community-operators 的子文件夹中提 pr 上传自己的 operator 的 csv 及 package 的 yaml 文件。

在集群上部署 operator 只需要定义 Subscription 文件，安装好 OLM 后 apply sub 文件就可以部署了。

## operator Metering

通过 operator-sdk 生成的项目，自带了 metrics 接口。operator Metering 本质上是通过提供 metrics 接口，再通过该接口提供的数据整合监控数据，最终生成数据报告。

该项目主要定义了三个 CR：

ReportDataSources: 定义有哪些可用数据  
ReportQueries: 定义如何查询 ReportDataSources 中定义的数据  
Reports: 根据 ReportQueries 生成数据报告

