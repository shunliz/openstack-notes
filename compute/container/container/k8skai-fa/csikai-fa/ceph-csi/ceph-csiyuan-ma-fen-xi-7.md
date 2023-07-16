## ceph-csi源码分析（4）-rbd driver-controllerserver分析

当ceph-csi组件启动时指定的driver type为rbd时，会启动rbd driver相关的服务。然后再根据controllerserver、nodeserver的参数配置，决定启动ControllerServer与IdentityServer，或NodeServer与IdentityServer。



基于tag v3.0.0

https://github.com/ceph/ceph-csi/releases/tag/v3.0.0



rbd driver分析将分为4个部分，分别是服务入口分析、controllerserver分析、nodeserver分析与IdentityServer分析。





