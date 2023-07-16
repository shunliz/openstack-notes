# ceph-csi源码分析（7）-rbd driver-IdentityServer分析 {#articleContentId}

当ceph-csi组件启动时指定的driver type为rbd时，会启动rbd driver相关的服务。然后再根据controllerserver、nodeserver的参数配置，决定启动ControllerServer与IdentityServer，或NodeServer与IdentityServer。

基于tag v3.0.0

[https://github.com/ceph/ceph-csi/releases/tag/v3.0.0](https://github.com/ceph/ceph-csi/releases/tag/v3.0.0)

rbd driver分析将分为4个部分，分别是服务入口分析、controllerserver分析、nodeserver分析与IdentityServer分析。

![](/assets/compute-container-k8s-cephcsi13633361.png)

这节进行IdentityServer分析，IdentityServer主要包括了GetPluginInfo（获取driver信息）、Probe（探测接口）、GetPluginCapabilities（获取driver能力）三个方法，将一一进行分析。



**IdentityServer分析**

**（1）GetPluginInfo**

**简介**

GetPluginInfo主要用于获取该ceph-csi driver的信息，如driver名称、版本等。



GetPluginInfo returns plugin information.



**（2）Probe**

**简介**

**Probe是一个探测接口，用于探测该driver是否启动/存活。**



**Probe returns empty response.**



Probe方法由liveness driver调用，liveness driver定时调用该方法，探测csi driver的存活，然后统计到prometheus metrics中。（liveness driver的相关代码相对简单，不再单独展开分析）



liveness driver调用Probe的有关代码位于internal/liveness/liveness.go-getLiveness\(\)。

被sidecar容器liveness调用，探测csi组件健康存活情况。



#### （3）GetPluginCapabilities

##### 简介

GetPluginCapabilities用于获取driver的能力。

GetPluginCapabilities returns available capabilities of the rbd driver.



**总结**

这节分析了GetPluginInfo、Probe、GetPluginCapabilities方法，作用分别如下：



GetPluginInfo：用于获取该ceph-csi driver的信息，如driver名称、版本等。

Probe：一个探测接口，用于探测该driver是否启动。

GetPluginCapabilities：用于获取driver的能力。





