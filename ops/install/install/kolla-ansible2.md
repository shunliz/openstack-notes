# kolla-ansible部署openstack

**设计规划**

目前设计了2类角色, ceph和nova. 只要不是ceph集群的节点, 则都是nova, 需要承担计算服务,控制节点和网络节点目前由ceph{01…03}担任. 接双线.

| vlan | 名称 | 网段\(CIDR标记\) | 用途 | 设备 | 备注 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 1031-1060 | os-tenant | 用户自定义 | 项目私有网络 | 计算及网络节点所在的二层交换机 | 31个私有网络, 应该够了, 不然今后扩展为900-1030吧. |
| 1031 | os-wuhan31 | 100.100.31.0/24 | 业务区\(wuhan31\)主机网络 | 计算及网络节点所在的二层交换机 | 本集群不需要. 写着是为了避免用错. |
| 33 | os-extnet | 192.168.33.0/24 | 浮动IP网. 私有网络NAT. | 所有节点的二层交换机, 三层交换机. | 让私有网络可以访问外界, 或者从外界进入\(绑定浮动IP后\) |
| 34-37 | os-pubnet | 192.168.34.0/24 - 192.168.37.0/24 | 直通内网 | 所有节点的二层交换机, 三层交换机 | 一般作为公共的出口网络. |

**IP及主机名规划**

网关为 100.100.31.1

```
127.0.0.1 localhost
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

100.100.31.254 cloud-wuhan31.***.org

100.100.31.201 wuhan31-ceph01.v3.os wuhan31-ceph01
100.100.31.202 wuhan31-ceph02.v3.os wuhan31-ceph02
100.100.31.203 wuhan31-ceph03.v3.os wuhan31-ceph03
100.100.31.102 wuhan31-nova01.v3.os wuhan31-nova01
100.100.31.103 wuhan31-nova02.v3.os wuhan31-nova02
```

## 一，基础环境准备

1，环境准备

| 系统 | ip | 主机名 | 角色 |
| :--- | :--- | :--- | :--- |
| centos7.4 | 100.100.31.201 | wuhan31-ceph01.v3.os | ceph01、kolla-ansible |
| centos7.4 | 100.100.31.202 | wuhan31-ceph02.v3.os | ceph02 |
| centos7.4 | 100.100.31.203 | wuhan31-ceph03.v3.os | ceph03 |
| centos7.4 | 100.100.31.101 | wuhan31-nova01.v3.os | nova01 |
| centos7.4 | 100.100.31.102 | wuhan31-nova02.v3.os | nova01 |

ip和主机名写入到/etc/hosts里

2，修改主机名

```
hostnamectl set-hostname wuhan31-ceph01.v3.os
hostnamectl set-hostname wuhan31-ceph02.v3.os
hostnamectl set-hostname wuhan31-ceph03.v3.os
```

3，关闭防火墙、selinux

```
systemctl stop firewalld
systemctl disable firewalld
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
setenforce 0
```

4，配置yum源：

```
修改yum源为公司内部源.

包括centos的cloud和ceph的mimic源:

curl -v http://mirrors.***.org/repo/centos7.repo > /etc/yum.repos.d/CentOS-Base.repo
curl -v http://mirrors.***.org/repo/cloud.repo > /etc/yum.repos.d/cloud.repo
yum makecache
```

5，统一网卡名称

```
[root@localhost network-scripts]# cat ifcfg-bond0
DEVICE=bond0
BOOTPROTO=static
TYPE=bond
ONBOOT=yes
IPADDR=100.100.31.203
NETMASK=255.255.255.0
GATEWAY=100.100.31.1
DNS1=192.168.55.55
USERCTL=no
BONDING_MASTER=yes
BONDING_OPTS="miimon=200 mode=1"
[root@localhost network-scripts]# cat ifcfg-em1
TYPE=Ethernet
BOOTPROTO=none
DEVICE=em1
ONBOOT=yes
MASTER=bond0
SLAVE=yes

[root@localhost network-scripts]# cat ifcfg-em2
TYPE=Ethernet
BOOTPROTO=none
DEVICE=em2
ONBOOT=yes
MASTER=bond0
SLAVE=yes
[root@localhost network-scripts]#
```

所有设备网卡名称使用bond0

6，安装docker

配置docker yum源

```
cat > /etc/yum.repos.d/docker.repo <<EOF
[docker]
name=docker
baseurl=https://download.docker.com/linux/centos/7/x86_64/stable

enabled=1
gpgcheck=0
EOF
```

然后安装docker-ce

```
curl http://mirrors.***.org/repo/docker.repo > /etc/yum.repos.d/docker.repo
yum install docker-ce
```

配置私有仓库

```
mkdir /etc/docker
cat > /etc/docker/daemon.json <<EOF
{
"registry-mirrors": ["http://mirrors.***.org:5000"]
}
EOF
```

启动服务

```
systemctl enable docker
systemctl start docker
```

7，安装所需软件

所有的节点需要安装:

yum install ceph python-pip -y

调试辅助工具，为了方便调试, 建议安装补全脚本.

yum install bash-completion-extras libvirt-bash-completion net-tools bind-utils sysstat iftop nload tcpdump htop -y

8，安装kolla-ansible

安装pip.

yum install python-pip -y

安装kolla-ansible所需的依赖软件:

yum install ansible python2-setuptools python-cryptography python-openstackclient -y

使用pip安装kolla-ansible:

pip install kolla-ansible

\_\*\*注意：

\_\*\*如果出现\`requests 2.20.0 has requirement idna&lt;2.8,&gt;=2.5, but you'll have idna 2.4 which is incompatible.\`错误，则强制更新requets库

\_\*\*pip install --ignore-installed requests

\_\*\*同样，出现Cannot uninstall 'PyYAML'. It is a distutils installed project and thus we cannot accurately determine which files belong to it which would lead to only a partial uninstall.错误，强制更新

\_\*\*sudo pip install --ignore-installed PyYAML

\_\*\*注：步骤1-7,9所有节点操作，步骤8在部署节点操作（这里用wuhan32-ceph01）

## 二，部署ceph集群

1，配置ceph用户远程登录

所有ceph节点操作。\(以下公钥根据自己机器实际情况填下，这一步的目的是能让ceph用户通过key登录系统\)

```
ssh-keygen -t rsa  //一路回车
usermod -s /bin/bash ceph
mkdir ~ceph/.ssh/
cat >> ~ceph/.ssh/authorized_keys << EOF
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDW6VghEC1cUrTZ6TfI9XcOEJZShkoL5YqtHBMtm2iZUnw8Pj6S3S1TCwKfdY0m+kInKlfZhoFCw3Xyee9XY7ZwPX6IEnixZMqO9EpC58LfxH841lw6xC0HesfF0QwWs+EVs5I1RwCN+Zoz2NPfu8RH30LHhBoSQpm75vRkF2trEbdtEI/kuzysO+73oF7R42lGJtgJtFbzLQSO2Vp/Xo7jdD/tdD/gcEsPniSPP3vFQg4EuSafdwxnJFuAxLAMCK+K1SQg7eNqboWYGhSWjOy39bTCZjieXOyNehPTVoqn3/qyC88c7D0PEbvTYxbNkuFU2MM7x9/k+ZGyvYnpex4t root@wuhan31-ceph01.os
EOF
cat >> ~/.ssh/authorized_keys << EOF
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDW6VghEC1cUrTZ6TfI9XcOEJZShkoL5YqtHBMtm2iZUnw8Pj6S3S1TCwKfdY0m+kInKlfZhoFCw3Xyee9XY7ZwPX6IEnixZMqO9EpC58LfxH841lw6xC0HesfF0QwWs+EVs5I1RwCN+Zoz2NPfu8RH30LHhBoSQpm75vRkF2trEbdtEI/kuzysO+73oF7R42lGJtgJtFbzLQSO2Vp/Xo7jdD/tdD/gcEsPniSPP3vFQg4EuSafdwxnJFuAxLAMCK+K1SQg7eNqboWYGhSWjOy39bTCZjieXOyNehPTVoqn3/qyC88c7D0PEbvTYxbNkuFU2MM7x9/k+ZGyvYnpex4t root@wuhan31-ceph01.os
EOF
cat > /etc/sudoers.d/ceph <<EOF
ceph ALL = (root) NOPASSWD:ALL
Defaults:ceph !requiretty
EOF
chown -R ceph:ceph ~ceph/.ssh/
chmod -R o-rwx ~ceph/.ssh/
```

2，创建ceph集群

在部署节点操作

安装部署工具ceph-deploy

yum install ceph-deploy -y

```
mkdir ~ceph/ceph-deploy
cd ~ceph/ceph-deploy
ceph-deploy new wuhan31-ceph{01..03}.os
```

编辑配置文件ceph.conf

```
vim ceph.conf
[global]
fsid = 567be343-d631-4348-8f9d-2f18be36ce74
mon_initial_members = wuhan31t-ceph01, wuhan31-ceph02,wuhan31-ceph03
mon_host = wuhan31-ceph01,wuhan31-ceph02,wuhan31-ceph03
mon_addr = 100.100.31.201:6789,00.100.31.202:6789,00.100.31.203:6789
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx
filestore_xattr_use_omap = true
mon_allow_pool_delete = 1

[osd]
osd_client_message_size_cap = 524288000
osd_deep_scrub_stride = 131072
osd_op_threads = 2
osd_disk_threads = 1
osd_mount_options_xfs = "rw,noexec,nodev,noatime,nodiratime,nobarrier"
osd_recovery_op_priority = 1
osd_recovery_max_active = 1
osd_max_backfills = 1
osd-recovery-threads=1

[client]
rbd_cache = true
rbd_cache_size = 1073741824
rbd_cache_max_dirty = 134217728
rbd_cache_max_dirty_age = 5
rbd_cache_writethrough_until_flush = true
rbd_concurrent_management_ops = 50
rgw frontends = civetweb port=7480
```

然后创建初始节点:

```
ceph-deploy mon create-initial
ceph-deploy admin wuhan31-ceph01 wuhan31-ceph02,wuhan31-ceph03
# 可选: 允许ceph用户使用admin keyring.
sudo setfacl -m u:ceph:r /etc/ceph/ceph.client.admin.keyring
```

创建mgr:

ceph-deploy mgr create wuhan31-ceph01 wuhan31-ceph02,wuhan31-ceph03

添加osd

这里是复用的硬盘, 故需要先zap disk:（我的机器磁盘是sdb到sdk）

```
ceph-deploy disk zap wuhan31-ceph01 /dev/sd{b..k}
ceph-deploy disk zap wuhan31-ceph02 /dev/sd{b..k}
ceph-deploy disk zap wuhan31-ceph03 /dev/sd{b..k}
```

可以使用如下的脚本批量添加osd:

```
for dev in /dev/sd{b..k}; do ceph-deploy osd create --data "$dev"wuhan31-ceph01 || break; done
for dev in /dev/sd{b..k}; do ceph-deploy osd create --data "$dev" wuhan31-ceph02 || break; done
for dev in /dev/sd{b..k}; do ceph-deploy osd create --data "$dev" wuhan31-ceph03 || break; done
```

如果期间遇到了错误, 可以单独继续执行。

3，创建pools

在部署节点操作

创建openstack 所需的pools:

计算方法： https://ceph.com/pgcalc/

根据大小设置不同的pg数. 由于目前只有3\*10个osd. 故初步定pg数量如下: 后期按需扩大



