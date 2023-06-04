ovn-kubernetes提供了一个ovs OVN网络插件，支持underlay和overlay两种模式。

* underlay：容器运行在虚拟机中，而ovs则运行在虚拟机所在的物理机上，OVN将容器网络和虚拟机网络连接在一起
* overlay：OVN通过logical overlay network连接所有节点的容器，此时ovs可以直接运行在物理机或虚拟机上

## Overlay模式 {#67c38bb1923163122f1f035740720b16}

![](/assets/network-virtualnet-ovnk8s1.png)**配置master**

```
ovs-vsctl set Open_vSwitch . external_ids:k8s-api-server="127.0.0.1:8080"
ovn-k8s-overlay master-init
--cluster-ip-subnet="192.168.0.0/16"
--master-switch-subnet="192.168.1.0/24"
--node-name="kube-master"
```

### 配置Node {#b0baf4205e7b78b38fa530d8d17e3b3e}

```
ovs-vsctl set Open_vSwitch .
external_ids:k8s-api-server="$K8S_API_SERVER_IP:8080"
ovs-vsctl set Open_vSwitch .
external_ids:k8s-api-server="https://$K8S_API_SERVER_IP"
external_ids:k8s-ca-certificate="$CA_CRT"
external_ids:k8s-api-token="$API_TOKEN"
ovn-k8s-overlay minion-init
--cluster-ip-subnet="192.168.0.0/16"
--minion-switch-subnet="192.168.2.0/24"
--node-name="kube-minion1"
```

### 配置网关Node \(可以用已有的Node或者单独的节点\) {#2dd43e300c50362ea1ce514ecadaee6b}

选项一：外网使用单独的网卡`eth1`

```
ovs-vsctl set Open_vSwitch .
external_ids:k8s-api-server="$K8S_API_SERVER_IP:8080"
ovn-k8s-overlay gateway-init
--cluster-ip-subnet="192.168.0.0/16"
--physical-interface eth1
--physical-ip 10.33.74.138/24
--node-name="kube-minion2"
--default-gw 10.33.74.253
```

选项二：外网网络和管理网络共享同一个网卡，此时需要将该网卡添加到网桥中，并迁移IP和路由

```
# attach eth0 to bridge breth0 and move IP/routes
ovn-k8s-util nics-to-bridge eth0
# initialize gateway
ovs-vsctl set Open_vSwitch .
external_ids:k8s-api-server="$K8S_API_SERVER_IP:8080"
ovn-k8s-overlay gateway-init
--cluster-ip-subnet="$CLUSTER_IP_SUBNET"
--bridge-interface breth0
--physical-ip "$PHYSICAL_IP"
--node-name="$NODE_NAME"
--default-gw "$EXTERNAL_GATEWAY"
# Since you share a NIC for both mgmt and North-South connectivity, you will
# have to start a separate daemon to de-multiplex the traffic.
ovn-k8s-gateway-helper --physical-bridge=breth0 --physical-interface=eth0
--pidfile --detach
```

### 启动ovn-k8s-watcher {#0c5015be3a45746f8576663e48d6026d}

ovn-k8s-watcher监听Kubernetes事件，并创建逻辑端口和负载均衡。

```
ovn-k8s-watcher 
  --overlay 
  --pidfile 
  --log-file 
  -vfile:info 
  -vconsole:emer 
  --detach
```

### CNI插件原理 {#995f45be957e0e35525b697388cf50a0}

#### ADD操作 {#5e4234df58dbfccab29917de49141034}

* 从
  `ovn`
  annotation获取ip/mac/gateway
* 在容器netns中配置接口和路由
* 添加ovs端口

```
ovs-vsctl add-port br-int veth_outside
--set interface veth_outside
external_ids:attached_mac=mac_address
external_ids:iface-id=namespace_pod
external_ids:ip_address=ip_address
```

#### DEL操作 {#03e9656578e9d263883c0782791759ee}

```
ovs-vsctl del-port br-int port
```



