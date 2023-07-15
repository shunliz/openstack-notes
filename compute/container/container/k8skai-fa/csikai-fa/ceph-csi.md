概述

下面的分析是k8s通过ceph-csi（csi plugin）接入ceph存储（csi相关组件的分析以rbd为例进行分析），对csi系统结构、所涉及的k8s对象与组件进行了简单的介绍，以及k8s对存储进行相关操作的流程分析，存储相关操作包括了存储创建、存储扩容、存储挂载、解除存储挂载以及存储删除操作。



csi系统结构

这是一张k8s csi的系统架构图，图中所画的组件以及k8s对象，接下来会一一进行分析。

![](/assets/compute-container-k8s-cephcsi11.png)csi简介

CSI是Container Storage Interface（容器存储接口）的简写。



CSI的目的是定义行业标准“容器存储接口”，使存储供应商（SP）能够开发一个符合CSI标准的插件并使其可以在多个容器编排（CO）系统中工作。CO包括Cloud Foundry, Kubernetes, Mesos等。



CSI组件一般采用容器化部署，减少了环境依赖。



涉及k8s对象

1. PersistentVolume

持久存储卷，集群级别资源，代表了存储卷资源，记录了该存储卷资源的相关信息。



回收策略

（1）retain：保留策略，当删除PVC的时候，PV与外部存储资源仍然存在。



（2）delete：删除策略，当与pv绑定的pvc被删除的时候，会从k8s集群中删除PV对象，并执行外部存储资源的删除操作。



（3）resycle（已废弃）



pv状态迁移

available --&gt; bound --&gt; released



2. PersistentVolumeClaim

持久存储卷声明，namespace级别资源，代表了用户对于存储卷的使用需求声明。



示例：



```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test
  namespace: test
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  storageClassName: csi-cephfs-sc
  volumeMode: Filesystem
```



