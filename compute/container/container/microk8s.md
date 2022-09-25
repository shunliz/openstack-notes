# ubuntu中使用MicroK8s

**Ubuntu 22.0.4中安装过程中提示是否要安装microk8s.**

```
使用Microk8s在本地启动一个单节点k8s集群。
启动microk8s自带的几个插件，包括DNS和Dashboard。
运行一个nginx应用查看效果。
```

```
sudo microk8s.kubectl get nodes
NAME        STATUS     ROLES    AGE   VERSION
mymachine   NotReady   <none>   18d   v1.23.6-2+2a84a218e3cd52
```

正常情况下应该看到的是当前状态为Ready这样的输出，显示我们的集群中有一个k8s工作节点。但是

**如果你使用的机器不能够科学上网的话，可能节点的状态会为NotReady**

。接下来先介绍如何简化kubectl命令使用，接着介绍节点NotReady情况下要怎么修复。

## 简化kubectl命令

\(1\)首先解决必须要sudo才能执行microk8s命令的问题。运行下面的命令，将你当前的用户加到 microk8s 用户组内：

```
sudo usermod -a -G microk8s ubuntu
sudo chown -f -R ubuntu ~/.kube
newgrp microk8s//重新加载用户组
```

\(2\)然后解决每次kubectl命令前面都必须要加上microk8s的问题，我们给microk8s.kubectl取别名为mkubectl:

```
sudo snap alias microk8s.kubectl mkubectl
mkubectl get nodes//可以简单使用
```

## 解决科学上网问题

该问题NotReady的原因是由于谷歌服务器被墙，导致有一些镜像拉不到。通过下面的几条命令就可以看到为什么集群的状态不健康了：

```
mkubectl get pods -n kube-system
mkubectl describe pod calico-node-q424c -n kube-system
```

```
(1)拉取镜像
sudo docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.1
(2)修改标签
sudo docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.1 k8s.gcr.io/pause:3.1
(3)删除原镜像
sudo docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.1
```

当本地Docker镜像仓库中有了k8s.gcr.io/pause:3.1这个镜像之后，Pod却仍然无法获取镜像，这是怎么回事？原来Microk8s使用的CRE是containerd，我们需要再将Docker镜像仓库里的镜像放到Microk8s使用的镜像仓库里去：

```
sudo docker save k8s.gcr.io/pause:3.1 > pause.tar
microk8s ctr image import pause.tar
```

这下再看pod状态已经都正常了：

## 启用插件

使用MicroK8s其中最大的好处之一事实上是也支持各种各样的插件和扩展。更重要的是它们是开箱即用的，用户仅仅需要启动它们。通过运行 microk8s.status命令检查出扩展的完整列表。

```
sudo microk8s.status
```

### 开启Dashboard

```
sudo microk8s enable dashboard
```

**拉取缺失的镜像**

```
sudo microk8s kubectl get pods -n kube-system
mkubectl describe pod metrics-server-679c5f986d-fnpq2 -n kube-system
发现缺少镜像k8s.gcr.io/metrics-server/metrics-server:v0.5.2
sudo docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/metrics-server:v0.5.2
sudo docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/metrics-server:v0.5.2 k8s.gcr.io/metrics-server/metrics-server:v0.5.2
sudo docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/metrics-server:v0.5.2
sudo docker save  k8s.gcr.io/metrics-server/metrics-server:v0.5.2 > metrics-server.tar
microk8s ctr image import metrics-server.tar

```

**获取Token**:

```
token=$(microk8s kubectl -n kube-system get secret | grep default-token | cut -d " " -f1)
microk8s kubectl -n kube-system describe secret $token

```

**获取dashboard的ClusterIP**

```
microk8s kubectl get svc -n kube-system
```

### [外网](https://so.csdn.net/so/search?q=%E5%A4%96%E7%BD%91&spm=1001.2101.3001.7020)访问Dashboard

**一、编辑aa.yaml文件**

```
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-zdy
  namespace: kube-system
spec:
  type: NodePort
  ports:
    - port: 443
      nodePort: 32000
      targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
```

**启动**

```
sudo microk8s kubectl create -f aa.yaml
```

获取token后，通过浏览器https://192.168.1.5:32000/访问。

登录填写刚才获取的token，开始访问dashboard.

