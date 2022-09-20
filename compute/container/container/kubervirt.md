Kubervirt是一套通过K8S管理虚拟机的方案。

```
  +---------------------+
  | KubeVirt            |
~~+---------------------+~~
  | Orchestration (K8s) |
  +---------------------+
  | Scheduling (K8s)    |
  +---------------------+
  | Container Runtime   |
~~+---------------------+~~
  | Operating System    |
  +---------------------+
  | Virtual(kvm)        |
~~+---------------------+~~
  | Physical            |
  +---------------------+
```

![](/assets/compute-container-k8s-kubevirt1.png)

[https://www.gremwell.com/node/155](https://www.gremwell.com/node/155)libvirt 管理Esxi

[https://github.com/kubevirt/kubevirt](https://github.com/kubevirt/kubevirt)

