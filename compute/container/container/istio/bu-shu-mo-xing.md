## 集群模型 {#cluster-models}

### 单一集群 {#single-cluster}

![](/assets/compute-container-k8s-istio-depmodel1.png)

单一集群部署提供了简单性，但缺少更多的功能，例如，故障隔离和故障转移。 如果您需要高可用性，则应使用多个集群。



### 多集群 {#multiple-clusters}

与单一集群部署相比其具备以下更多能力：

* 故障隔离和故障转移：当`cluster-1`下线，业务将转移至`cluster-2`。
* 位置感知路由和故障转移：将请求发送到最近的服务。
* 多种[控制平面](https://istio.io/latest/zh/docs/ops/deployment/deployment-models/#control-plane-models)模型：支持不同级别的可用性。
* 团队或项目隔离：每个团队仅运行自己的集群。![](/assets/compute-container-k8s-istio-depmodel2.png)

## 网络模型 {#network-models}

### 单一网络 {#single-network}

在最简单的情况下，服务网格在单个完全连接的网络上运行。 在单一网络模型中，工作负载实例都可以直接相互访问，而无需 Istio 网关。

单一网络模型允许 Istio 以统一的方式在网格上配置服务使用者，从而能够直接处理工作负载实例。

![](/assets/compute-container-k8s-istio-depmodel3.png)

### 多网络 {#multiple-networks}

您可以配置单个服务网格跨多个网络，这样的配置称为**多网络**。

多网络模型提供了单一网络之外的以下功能：

* **服务端点**
  范围的 IP 或 VIP 重叠
* 跨越管理边界
* 容错能力
* 网络地址扩展
* 符合网络分段要求的标准

在此模型中，不同网络中的工作负载实例只能通过一个或多个 Istio 网关相互访问。Istio 使用**分区服务发现**为消 费者提供服务端点的不同视图。该视图取决于消费者的网络。

![](/assets/compute-container-k8s-istio-depmodel4.png)  


参考

https://istio.io/latest/zh/docs/ops/deployment/deployment-models/

