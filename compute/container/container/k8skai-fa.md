# pod创建流程

![](/assets/compute-container-k8s-dev-podcrd1.png)

**基本概念**首先介绍一下本节所涉及到的基本概念。

* CRD \(Custom Resource Definition\)：允许用户自定义 Kubernetes 资源，是一个类型；

* CR \(Custom Resourse\)：CRD 的一个具体实例；

* webhook：它本质上是一种 HTTP 回调，会注册到 apiserver 上。在 apiserver 特定事件发生时，会查询已注册的 webhook，并把相应的消息转发过去。

按照处理类型的不同，一般可以将其分为两类：一类可能会修改传入对象，称为 mutating webhook；一类则会只读传入对象，称为 validating webhook。

* 工作队列：controller 的核心组件。它会监控集群内的资源变化，并把相关的对象，包括它的动作与 key，例如 Pod 的一个 Create 动作，作为一个事件存储于该队列中；

* controller：它会循环地处理上述工作队列，按照各自的逻辑把集群状态向预期状态推动。不同的 controller 处理的类型不同，比如 replicaset controller 关注的是副本数，会处理一些 Pod 相关的事件；

* operator：operator 是描述、部署和管理 kubernetes 应用的一套机制，从实现上来说，可以将其理解为 CRD 配合可选的 webhook 与 controller 来实现用户业务逻辑，即 operator = CRD + webhook + controller。



**常见的 operator 工作模式**

![](/assets/compute-container-k8s-dev-operator1.png)

工作流程：

1. 用户创建一个自定义资源 \(CRD\)；

2. apiserver 根据自己注册的一个 pass 列表，把该 CRD 的请求转发给 webhook；

3. webhook 一般会完成该 CRD 的缺省值设定和参数检验。webhook 处理完之后，相应的 CR 会被写入数据库，返回给用户；

4. 与此同时，controller 会在后台监测该自定义资源，按照业务逻辑，处理与该自定义资源相关联的特殊操作；

5. 上述处理一般会引起集群内的状态变化，controller 会监测这些关联的变化，把这些变化记录到 CRD 的状态中。



