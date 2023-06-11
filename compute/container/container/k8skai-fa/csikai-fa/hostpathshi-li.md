## 实例 {#实例}

csi-driver-host-path 是社区实现的一个 CSI 插件的示例，它以 hostpath 为后端存储，kubernetes 通过这个 CSI 插件 driver 来对接 hostpath ，管理本地 Node 节点上的存储卷。我们以这个为例子，来看看如何编写一个csi插件。

### 目录结构 {#目录结构}

```
├── hostpath
│   ├── controllerserver.go
│   ├── nodeserver.go
│   ├── nodeserver_test.go
│   ├── hostpath.go
│   └── utils.go
│   └── identityserver.go
```

* controllerserver.go主要是实现controller的rpc接口

* nodeserver.go主要是实现node的rpc接口

* utils.go基本公用函数
* hostpath.go入口，启动rpc服务，只要把编写好的 gRPC Server 注册给 CSI，它就可以响应来自 External Components 的 CSI 请求了。
* identityserver.go身份验证的rpc接口



