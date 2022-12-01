### OpenKruise

OpenKruise 由 Alibaba 开源，是[Kubernetes](https://so.csdn.net/so/search?q=Kubernetes&spm=1001.2101.3001.7020)的一个标准扩展，它可以配合原生 Kubernetes 使用，并为管理应用容器、`sidecar`、镜像分发等方面提供更加强大和高效的能力。

OpenKruise 是面向自动化场景的 Kubernetes 应用负载扩展控制器。而 Kruise 是 OpenKruise 项目的核心，它是一组控制器，可在应用程序工作负载管理上扩展和补充 Kubernetes 核心控制器。

官网：openkruise.io 最新版本：0.8.0

github地址：https://github.com/openkruise/kruise



核心功能：

```
原地升级：原地升级是一种可以避免删除、新建 Pod 的升级镜像能力。它比原生 Deployment/StatefulSet 的重建 Pod 升级更快、更高效，
          并且避免对 Pod 中其他不需要更新的容器造成干扰

Sidecar 管理：支持在一个单独的 CR 中定义 sidecar 容器，OpenKruise 能够帮你把这些 Sidecar 容器注入到所有符合条件的 Pod 中。
              这个过程和 Istio 的注入很相似，但是你可以管理任意你关心的 Sidecar

跨多可用区部署：定义一个跨多个可用区的全局 workload、容器，OpenKruise 会帮你在每个可用区创建一个对应的下属 workload。
                你可以统一管理 workload 的副本数、版本、甚至针对不同可用区采用不同的发布策略

镜像预热：支持用户指定在任意范围的节点上下载镜像


```

* CRD列表：

```
CloneSet    提供更加高效、确定可控的应用管理和部署能力，支持优雅原地升级、指定删除、发布顺序可配置、并行/灰度发布等丰富的策略，可以满足更多样化的应用场景

Advanced StatefulSet    基于原生 StatefulSet 之上的增强版本，默认行为与原生完全一致，在此之外提供了原地升级、并行发布（最大不可用）、发布暂停等功能

SidecarSet  对 sidecar 容器做统一管理，在满足 selector 条件的 Pod 中注入指定的 sidecar 容器

UnitedDeployment    通过多个 subset workload 将应用部署到多个可用区

BroadcastJob    配置一个 job，在集群中所有满足条件的 Node 上都跑一个 Pod 任务

Advanced DaemonSet  基于原生 DaemonSet 之上的增强版本，默认行为与原生一致，在此之外提供了灰度分批、按 Node label 选择、暂停、热升级等发布策略

AdvancedCronJob     一个扩展的 CronJob 控制器，目前 template 模板支持配置使用 Job 或 BroadcastJob

ImagePullJob        支持用户指定在任意范围的节点上下载镜像
```

### 安装 OpenKruise

OpenKruise 要求 Kubernetes 版本高于 1.13+，注意在 1.13 和 1.14 版本中必须先在 Kube-ApiServer 中打开`CustomResourceWebhookConversion`feature-gate。

* 通过 helm 安装：

建议采用 helm v3.1+ 来安装 Kruise，

```
# Kubernetes 1.13 或 1.14 版本

helm install kruise https://github.com/openkruise/kruise/releases/download/v0.8.0/kruise-chart.tgz --disable-openapi-validation


# Kubernetes 1.15 和更高版本

helm install kruise https://github.com/openkruise/kruise/releases/download/v0.8.0/kruise-chart.tgz


```





