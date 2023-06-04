ovn-kubernetes提供了一个ovs OVN网络插件，支持underlay和overlay两种模式。

* underlay：容器运行在虚拟机中，而ovs则运行在虚拟机所在的物理机上，OVN将容器网络和虚拟机网络连接在一起
* overlay：OVN通过logical overlay network连接所有节点的容器，此时ovs可以直接运行在物理机或虚拟机上

## Overlay模式 {#67c38bb1923163122f1f035740720b16}



