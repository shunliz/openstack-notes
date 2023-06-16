**Extension以及Port binding简述**

Extension 是 Neutron 项目中对基本资源实现属性扩展的手段，而 Portbinding 则是 Neutron 在早期即引入的一个扩展模块，可以实现对 Port 资源的属性扩展。该扩展模块在 neutron 及 Agent 将 port 和实际网卡资源绑定时发挥重要作用，使得管理员可以人工指定或者获取 port 的物理绑定信息，其属性的正确与否会影响到 server（虚拟机、baremeta）的能否成功启动。

**Port binding 的属性**

Port binding 扩展定义的属性主要包括 vnic\_type、vif\_type、vif\_details、host\_id、profile ,这些属性都可以在创建 port 的 REST API接口中可以直接指定。

一个创建 port 的 REST API body 实例如下：

POST      /v2.0/ports

```js
{
    "port": {
        "binding:host_id": "4df8d9ff-6f6f-438f-90a1-ef660d4586ad",
        "binding:profile": {
            "local_link_information": [
                {
                    "port_id": "Ethernet3/1",
                    "switch_id": "0a:1b:2c:3d:4e:5f",
                    "switch_info": "switch1"
                }
            ]
        },
        "binding:vnic_type": "baremetal",
        "device_id": "d90a13da-be41-461f-9f99-1dbcf438fdf2",
        "device_owner": "baremetal:none",
        "dns_domain": "my-domain.org.",
        "dns_name": "myport",
        "qos_policy_id": "29d5e02e-d5ab-4929-bee4-4a9fc12e22ae"
    }
}
```

关于每个属性的具体含义、类型与取值范围如下表：![](/assets/network-virtualnetwork-netron-code1.png)

**Port binding 的属性在Neutron的mechanism  driver中作用**

Neutron.plugin.ml2.plugin.Ml2plugin类方法 \_bind\_port\(\)中调用注册到mechanism manager的driver尝试对端口进行绑定。

```py
    def _bind_port(self, orig_context):
        # 构建一个新的port上下文结构
        port = orig_context.current
        orig_binding = orig_context._binding
        new_binding = models.PortBinding(
            host=orig_binding.host,
            vnic_type=orig_binding.vnic_type,
            profile=orig_binding.profile,
            vif_type=portbindings.VIF_TYPE_UNBOUND,
            vif_details=''
        )
        self._update_port_dict_binding(port, new_binding)
        new_context = driver_context.PortContext(
            self, orig_context._plugin_context, port,
            orig_context.network.current, new_binding, None,
            original_port=orig_context.original)

        # 以下开始使用注册的mechanism driver进行端口绑定
        self.mechanism_manager.bind_port(new_context)
        return new_context
```

在每个 mechanism driver 的 \_\_init\_\_\(\) 方法中会指明该 driver 是否支持端口包过滤特性，接下来会指定该 driver 支持的 vNIC 类型，ML2 的 mechanism driver 只会调用支持该端口指定vNIC 类型的driver来尝试执行绑定操作。

参考[https://blog.csdn.net/bestjie01/article/details/83628340](https://blog.csdn.net/bestjie01/article/details/83628340)

