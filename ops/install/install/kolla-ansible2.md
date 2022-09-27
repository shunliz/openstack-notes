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

计算方法： [https://ceph.com/pgcalc/](https://ceph.com/pgcalc/)

根据大小设置不同的pg数. 由于目前只有3\*10个osd. 故初步定pg数量如下: 后期按需扩大

```
images 32
volumes 256
vms 64
backups 128
```

在任意具备ceph admin权限的节点执行创建:

```
ceph osd pool create images 32
ceph osd pool create volumes 256
ceph osd pool create vms 64
ceph osd pool create backups 128
```

4，创建ceph客户端

在部署节点操作

创建客户端, 并赋予权限，以下信息写入脚本执行,或者直接执行

```
# 定义客户端
clients="client.cinder client.nova client.glance client.cinder-backup"
# 创建客户端.
for client in $clients; do
  ceph auth get-or-create "$client"
done



# 配置权限
ceph auth caps client.cinder mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=volumes, allow rwx pool=cinder-ssd, allow rwx pool=vms, allow rwx pool=images'
ceph auth caps client.nova mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=volumes, allow rwx pool=cinder-ssd, allow rwx pool=vms, allow rwx pool=images'
ceph auth caps client.glance mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=images'
ceph auth caps client.cinder-backup mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=backups'
# 导出
for client in $clients; do
  ceph auth export "$client" -o /etc/ceph/ceph."$client".keyring
done
```

定义创建客户端：

```
ceph auth get-or-create client.cinder
ceph auth get-or-create client.nova
ceph auth get-or-create client.glance
ceph auth get-or-create client.cinder-backup
```

配置权限

```
ceph auth caps client.cinder mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=volumes, allow rwx pool=cinder-ssd, allow rwx pool=vms, allow rwx pool=images'
ceph auth caps client.nova mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=volumes, allow rwx pool=cinder-ssd, allow rwx pool=vms, allow rwx pool=images'
ceph auth caps client.glance mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=images'
ceph auth caps client.cinder-backup mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=backups'
```

导出keyting

```
ceph auth export client.cinder -o /etc/ceph/ceph.client.cinder.keyring
ceph auth export client.nova -o /etc/ceph/ceph.client.nova.keyring
ceph auth export client.glance -o /etc/ceph/ceph.client.glance.keyring
ceph auth export client.cinder-backup -o /etc/ceph/ceph.client.cinder-backup.keyring
```

5，配置ceph插件dashboard

在部署节点操作

```
ceph mgr module enable dashboard
ceph config set mgr mgr/dashboard/ssl false
ceph config set mgr mgr/dashboard/server_address ::
ceph config set mgr mgr/dashboard/server_port 7000
ceph dashboard set-login-credentials 用户名 密码
```

## 三，kolla部署openstack

以下部署节点操作

1，编写配置

复制模板

复制kolla-ansible的模板, 这里是使用pip安装的:

必选: 复制配置模板

cp -ar /usr/share/kolla-ansible/etc\_examples/\* /etc/

2，生成密码

请务必完成 “复制模板” 的环节. 否则无法生成密码

执行如下命令即可

kolla-genpwd

globals.yml

编辑修改/etc/kolla/globals.yml

```
# 这里是openstack的版本信息. 这里选择rocky版本,source即源码安装, 因为这种方式的软件包最全. 如果为binary且为CentOS系统, 那么只有红帽提供的包, 有些不全.
kolla_install_type: "source"
openstack_release: "rocky"

# 如果有多个控制节点, 则启用高可用, 注意, vip(虚拟IP)必须为目前未用到的IP. 且和节点IP位于同一网段.
enable_haproxy: "yes"
kolla_internal_vip_address: "100.100.31.254"

# 这些fqdn需要在内网DNS和hosts文件同时做好解析.
kolla_internal_fqdn: "xiaoxuantest.***.org"
kolla_external_fqdn: "xiaoxuantest.***.org"

# 这里就是自定义配置的路径. 只在部署节点上.
node_custom_config: "/etc/kolla/config"

# 虚拟化类型, 如果是在虚拟机里做实验, 这里的类型需要改为qemu. 慢点就慢点.
# kvm类型需要CPU,主板和BIOS支持, 且BIOS启用了硬件虚拟化. 如果在计算节点无法安装kvm内核模块, 请根据dmesg报错排查.
nova_compute_virt_type: "kvm"

# 网络接口. 注意external必须为独立接口, 不然会导致节点断网.
neutron_external_interface: "eth1"
network_interface: "bond0"
api_interface: "bond0"
storage_interface: "bond0"
cluster_interface: "bond0"
tunnel_interface: "bond0"
# dns_interface: "eth"  # dns功能未集成, 后期自行研究吧.

# 网络虚拟化技术. 我们这里不使用openvswitch, 直接使用linuxbridge
neutron_plugin_agent: "linuxbridge"
enable_openvswitch: "no"
# 网络高可用, 就是创建多个agent: dhcp和l3(路由)
enable_neutron_agent_ha: "yes"
# 网络封装, 目前都是vlan, flat留着备用, 用于直接使用物理网卡.
neutron_type_drivers: "flat,vlan"
# 租户网络的隔离方式, 这里是vlan, 但是kolla不支持, 所以我们需要自己在node_custom_config这项对应的目录里加自定义配置.
neutron_tenant_network_types: "vlan"

# 网络插件
enable_neutron_lbaas: "yes"
enable_neutron_***aas: "yes"
enable_neutron_fwaas: "yes"

# elk集中日志管理
enable_central_logging: "yes"
# 启用debug模式, 日志很详细. 按需临时开启.
#openstack_logging_debug: "True"

# 忘了这里的用途... 可以关了试试, 如果其他组件有依赖会自动开的.
enable_kafka: "yes"
enable_fluentd: "yes"

# 这里是我们使用了外部的ceph, 不让kolla部署, 因为kolla部署时部分osd可能会出问题, 导致osd id顺序错位, 看着不方便. 而且后期从主机管理存储集群也别捏.
enable_ceph: "no"
glance_backend_ceph: "yes"
cinder_backend_ceph: "yes"
nova_backend_ceph: "yes"
gnocchi_backend_storage: "ceph"
enable_manila_backend_cephfs_native: "yes"

# 启用的功能.
#enable_ceilometer: "yes"
enable_cinder: "yes"
#enable_designate: "yes"
enable_destroy_images: "yes"
#enable_gnocchi: "yes"
enable_grafana: "yes"
enable_heat: "yes"
enable_horizon: "yes"
#enable_ironic: "yes"
#enable_ironic_ipxe: "yes"
#enable_ironic_neutron_agent: "yes"
#enable_kuryr: "yes"
#enable_magnum: "yes"
# enable_neutron_dvr
# enable_ovs_dpdk
#enable_nova_serialconsole_proxy: "yes"
#enable_octavia: "yes"
enable_redis: "yes"
#enable_trove: "yes"

# 其他配置
glance_backend_file: "no"
#designate_ns_record: "nova."
#ironic_dnsmasq_dhcp_range: "11.0.0.10,11.0.0.111"
openstack_region_name: "xiaoxuantest"
```

3，编写inventory文件

创建个目录, 用于编写inventory文件:

```
mkdir kolla-ansible
cp /usr/share/kolla-ansible/ansible/inventory/multinode kolla-ansible/inventory-xiaoxuantest
```

编辑后的inventory文件关键内容, 其他地方不变:

关键内容

```
[control]
wuhan31-ceph01
wuhan31-ceph02
wuhan31-ceph03

[network]
wuhan31-ceph01
wuhan31-ceph02
wuhan31-ceph03

[external-compute]
wuhan31-ceph01
wuhan31-ceph02
wuhan31-ceph03

[monitoring:children]
control

[storage:children]
control
```

4，ceph集成

网络

因为我们使用了vlan, 所以需要手动配置:

```
mkdir /etc/kolla/config/neutron
cat > /etc/kolla/config/neutron/ml2_conf.ini <<EOF
[ml2_type_vlan]
network_vlan_ranges = physnet0:1031:1060,physnet1

[linux_bridge]
physical_interface_mappings = physnet0:eth0,physnet1:eth1
EOF
```

Dashboard

创建虚拟机界面禁止默认创建新卷.

```
mkdir /etc/kolla/config/horizon/
cat > /etc/kolla/config/horizon/custom_local_settings <<EOF
LAUNCH_INSTANCE_DEFAULTS = {
  'create_volume': False,
}
EOF
```

直接贴上/etc/kolla/config/ 目录所有的文件

```
[root@wuhan32-ceph01 config]# ls -lR
.:
total 4
lrwxrwxrwx. 1 kolla kolla  19 Mar 11 17:06 ceph.conf -> /etc/ceph/ceph.conf
drwxr-xr-x. 4 kolla kolla 117 Mar 28 14:43 cinder
-rw-r--r--. 1 root  root   39 Mar 28 14:39 cinder.conf
drwxr-xr-x. 2 kolla kolla  80 Mar 11 17:18 glance
drwxr-xr-x. 2 root  root   35 Mar 19 11:21 horizon
drwxr-xr-x. 2 root  root   26 Mar 14 15:49 neutron
drwxr-xr-x. 2 kolla kolla 141 Mar 11 17:18 nova

./cinder:
total 8
lrwxrwxrwx. 1 kolla kolla  19 Mar 11 17:10 ceph.conf -> /etc/ceph/ceph.conf
drwxr-xr-x. 2 kolla kolla  81 Mar 11 17:18 cinder-backup
-rwxr-xr-x. 1 kolla kolla 274 Feb 26 16:47 cinder-backup.conf
drwxr-xr-x. 2 kolla kolla  40 Mar 11 17:18 cinder-volume
-rwxr-xr-x. 1 kolla kolla 534 Mar 28 14:38 cinder-volume.conf

./cinder/cinder-backup:
total 0
lrwxrwxrwx. 1 kolla kolla 43 Mar 11 17:18 ceph.client.cinder-backup.keyring -> /etc/ceph/ceph.client.cinder-backup.keyring
lrwxrwxrwx. 1 kolla kolla 36 Mar 11 17:18 ceph.client.cinder.keyring -> /etc/ceph/ceph.client.cinder.keyring

./cinder/cinder-volume:
total 0
lrwxrwxrwx. 1 kolla kolla 36 Mar 11 17:18 ceph.client.cinder.keyring -> /etc/ceph/ceph.client.cinder.keyring

./glance:
total 4
lrwxrwxrwx. 1 kolla kolla  36 Mar 11 17:18 ceph.client.glance.keyring -> /etc/ceph/ceph.client.glance.keyring
lrwxrwxrwx. 1 kolla kolla  19 Mar 11 17:07 ceph.conf -> /etc/ceph/ceph.conf
-rwxr-xr-x. 1 kolla kolla 138 Feb 27 11:55 glance-api.conf

./horizon:
total 4
-rw-r--r--. 1 root root 59 Mar 19 11:21 custom_local_settings

./neutron:
total 4
-rw-r--r--. 1 root root 141 Mar 14 15:49 ml2_conf.ini

./nova:
total 8
lrwxrwxrwx. 1 kolla kolla  36 Mar 11 17:18 ceph.client.cinder.keyring -> /etc/ceph/ceph.client.cinder.keyring
lrwxrwxrwx. 1 kolla kolla  34 Mar 11 17:18 ceph.client.nova.keyring -> /etc/ceph/ceph.client.nova.keyring
lrwxrwxrwx. 1 kolla kolla  19 Mar 11 17:07 ceph.conf -> /etc/ceph/ceph.conf
-rwxr-xr-x. 1 kolla kolla 101 Feb 27 17:28 nova-compute.conf
-rwxr-xr-x. 1 kolla kolla  39 Dec 24 16:52 nova-scheduler.conf
```

摘录config目录相关配置文件内容如下:

```
# find -type f -printf "=== FILE: %p ===\n" -exec cat {} \;
=== FILE: ./cinder/cinder-backup.conf ===
[DEFAULT]
backup_ceph_conf=/etc/ceph/ceph.conf
backup_ceph_user=cinder-backup
backup_ceph_chunk_size = 134217728
backup_ceph_pool=backups
backup_driver = cinder.backup.drivers.ceph
backup_ceph_stripe_unit = 0
backup_ceph_stripe_count = 0
restore_discard_excess_bytes = true
=== FILE: ./cinder/cinder-volume.conf ===
[DEFAULT]
enabled_backends=cinder-sas,cinder-ssd

[cinder-sas]
rbd_ceph_conf=/etc/ceph/ceph.conf
rbd_user=cinder
backend_host=rbd:volumes
rbd_pool=volumes
volume_backend_name=cinder-sas
volume_driver=cinder.volume.drivers.rbd.RBDDriver
rbd_secret_uuid=5b3ec4eb-c276-4cf2-a042-8ec906d05f69

[cinder-ssd]
rbd_ceph_conf=/etc/ceph/ceph.conf
rbd_user=cinder
backend_host=rbd:volumes
rbd_pool=cinder-ssd
volume_backend_name=cinder-ssd
volume_driver=cinder.volume.drivers.rbd.RBDDriver
rbd_secret_uuid=5b3ec4eb-c276-4cf2-a042-8ec906d05f69
=== FILE: ./glance/glance-api.conf ===
[glance_store]
default_store = rbd
stores = rbd
rbd_store_pool = images
rbd_store_user = glance
rbd_store_ceph_conf = /etc/ceph/ceph.conf
=== FILE: ./nova/nova-compute.conf ===
[libvirt]
images_rbd_pool=vms
images_type=rbd
images_rbd_ceph_conf=/etc/ceph/ceph.conf
rbd_user=nova
=== FILE: ./nova/nova-scheduler.conf ===
[DEFAULT]
scheduler_max_attempts = 100
=== FILE: ./neutron/ml2_conf.ini ===
[ml2_type_vlan]
network_vlan_ranges = physnet0:1000:1030,physnet1

[linux_bridge]
physical_interface_mappings = physnet0:eth0,physnet1:eth1

=== FILE: ./horizon/custom_local_settings ===

LAUNCH_INSTANCE_DEFAULTS = {
  'create_volume': False,
}

=== FILE: ./cinder.conf ===
[DEFAULT]
default_volume_type=standard
```

\*注意，./cinder/cinder-volume.conf里的rbd\_secret\_uuid=填写下面的这个值

```
[root@wuhan31-ceph01 ~]# grep cinder_rbd_secret_uuid /etc/kolla/passwords.yml 
cinder_rbd_secret_uuid: 1ae2156b-7c33-4fbb-a26a-c770fadc54b6
```

创建的连接如下

```
ceph.conf:
ln -s /etc/ceph/ceph.conf /etc/kolla/config/nova/
ln -s /etc/ceph/ceph.conf /etc/kolla/config/glance/
ln -s /etc/ceph/ceph.conf /etc/kolla/config/cinder/

keyring:

ln -s /etc/ceph/ceph.client.cinder-backup.keyring /etc/kolla/config/cinder/cinder-backup/
ln -s /etc/ceph/ceph.client.cinder.keyring /etc/kolla/config/cinder/cinder-backup/
ln -s /etc/ceph/ceph.client.cinder.keyring /etc/kolla/config/cinder/cinder-volume/
ln -s /etc/ceph/ceph.client.glance.keyring /etc/kolla/config/glance/
ln -s /etc/ceph/ceph.client.cinder.keyring /etc/kolla/config/nova/
ln -s /etc/ceph/ceph.client.nova.keyring /etc/kolla/config/nova/
```

5，部署openstack

预配节点环境

bootstrap也会安装不少东西, 后期可以考虑提前安装这些.

kolla-ansible -i kolla-ansible/inventory-xiaoxuantest bootstrap-servers

预检查

kolla-ansible -i kolla-ansible/inventory-xiaoxuantest prechecks

正式部署

kolla-ansible -i kolla-ansible/inventory-xiaoxuantest deploy

调整配置及重新部署

如果需要调整配置. 那么编辑globals.yml后, 然后运行reconfigure. 使用 -t 参数可以只对变动的模块进行调整.

kolla-ansible -i kolla-ansible/inventory-xiaoxuantest reconfigure -t neutron

kolla-ansible -i kolla-ansible/inventory-xiaoxuantest deploy -t neutron.

完成部署

这步主要是生成admin-openrc.sh.

kolla-ansible post-deploy

. /etc/kolla/admin-openrc.sh

初始demo

执行后会自动下载cirros镜像, 创建网络, 并创建一批测试虚拟机.

/usr/share/kolla-ansible/init-runonce

查询登录密码：

grep admin /etc/kolla/passwords.yml

如果没有内部源部署时间比较长，国外的资源下载很慢。

官方的故障排查指南: https://docs.openstack.org/kolla-ansible/latest/user/troubleshooting.html



部署成果后运行的容器如下：

```
[root@wuhan31-ceph01 ~]# docker ps -a
CONTAINER ID        IMAGE                                                 COMMAND                  CREATED             STATUS              PORTS               NAMES
d141ac504ec6        kolla/centos-source-grafana:rocky                     "dumb-init --single-…"   8 weeks ago         Up 8 weeks                              grafana
41e12c24ba7f        kolla/centos-source-horizon:rocky                     "dumb-init --single-…"   8 weeks ago         Up 8 weeks                              horizon
6989a4aeb33a        kolla/centos-source-heat-engine:rocky                 "dumb-init --single-…"   8 weeks ago         Up 8 weeks                              heat_engine
35665589b4a4        kolla/centos-source-heat-api-cfn:rocky                "dumb-init --single-…"   8 weeks ago         Up 8 weeks                              heat_api_cfn
f18c98468796        kolla/centos-source-heat-api:rocky                    "dumb-init --single-…"   8 weeks ago         Up 8 weeks                              heat_api
bc77f4d3c957        kolla/centos-source-neutron-metadata-agent:rocky      "dumb-init --single-…"   8 weeks ago         Up 8 weeks                              neutron_metadata_agent
7334b93c6564        kolla/centos-source-neutron-lbaas-agent:rocky         "dumb-init --single-…"   8 weeks ago         Up 8 weeks                              neutron_lbaas_agent
0dd7a55245c4        kolla/centos-source-neutron-l3-agent:rocky            "dumb-init --single-…"   8 weeks ago         Up 8 weeks                              neutron_l3_agent
beec0f19ec7f        kolla/centos-source-neutron-dhcp-agent:rocky          "dumb-init --single-…"   8 weeks ago         Up 8 weeks                              neutron_dhcp_agent
af6841ebc21e        kolla/centos-source-neutron-linuxbridge-agent:rocky   "dumb-init --single-…"   8 weeks ago         Up 8 weeks                              neutron_linuxbridge_agent
49dc0457445d        kolla/centos-source-neutron-server:rocky              "dumb-init --single-…"   8 weeks ago         Up 8 weeks                              neutron_server
677c0be4ab6b        kolla/centos-source-nova-compute:rocky                "dumb-init --single-…"   8 weeks ago         Up 8 weeks                              nova_compute
402b1e673777        kolla/centos-source-nova-novncproxy:rocky             "dumb-init --single-…"   8 weeks ago         Up 8 weeks                              nova_novncproxy
e35729b76996        kolla/centos-source-nova-consoleauth:rocky            "dumb-init --single-…"   8 weeks ago         Up 8 weeks                              nova_consoleauth
8b193f562e47        kolla/centos-source-nova-conductor:rocky              "dumb-init --single-…"   8 weeks ago         Up 8 weeks                              nova_conductor
885581445be0        kolla/centos-source-nova-scheduler:rocky              "dumb-init --single-…"   8 weeks ago         Up 8 weeks                              nova_scheduler
171128b7bcb7        kolla/centos-source-nova-api:rocky                    "dumb-init --single-…"   8 weeks ago         Up 8 weeks                              nova_api
8d7f3de2ad63        kolla/centos-source-nova-placement-api:rocky          "dumb-init --single-…"   8 weeks ago         Up 8 weeks                              placement_api
ab763320f268        kolla/centos-source-nova-libvirt:rocky                "dumb-init --single-…"   8 weeks ago         Up 8 weeks                              nova_libvirt
bbd4c3e2c961        kolla/centos-source-nova-ssh:rocky                    "dumb-init --single-…"   8 weeks ago         Up 8 weeks                              nova_ssh
80e7098f0bfb        kolla/centos-source-cinder-backup:rocky               "dumb-init --single-…"   8 weeks ago         Up 8 weeks                              cinder_backup
20e2ff43d0e1        kolla/centos-source-cinder-volume:rocky               "dumb-init --single-…"   8 weeks ago         Up 8 weeks                              cinder_volume
6caba29f7ce2        kolla/centos-source-cinder-scheduler:rocky            "dumb-init --single-…"   8 weeks ago         Up 8 weeks                              cinder_scheduler
3111622e4e83        kolla/centos-source-cinder-api:rocky                  "dumb-init --single-…"   8 weeks ago         Up 8 weeks                              cinder_api
2c011cfae829        kolla/centos-source-glance-api:rocky                  "dumb-init --single-…"   8 weeks ago         Up 8 weeks                              glance_api
be84e405afdd        kolla/centos-source-kafka:rocky                       "dumb-init --single-…"   8 weeks ago         Up 8 weeks                              kafka
09aef04ad59e        kolla/centos-source-keystone-fernet:rocky             "dumb-init --single-…"   8 weeks ago         Up 8 weeks                              keystone_fernet
2ba9e19844fd        kolla/centos-source-keystone-ssh:rocky                "dumb-init --single-…"   8 weeks ago         Up 8 weeks                              keystone_ssh
8eebe226b065        kolla/centos-source-keystone:rocky                    "dumb-init --single-…"   8 weeks ago         Up 8 weeks                              keystone
662d85c00a64        kolla/centos-source-rabbitmq:rocky                    "dumb-init --single-…"   8 weeks ago         Up 8 weeks                              rabbitmq
7d373ef0fdee        kolla/centos-source-mariadb:rocky                     "dumb-init kolla_sta…"   8 weeks ago         Up 8 weeks                              mariadb
ab9f5d612925        kolla/centos-source-memcached:rocky                   "dumb-init --single-…"   8 weeks ago         Up 8 weeks                              memcached
a728298938f7        kolla/centos-source-kibana:rocky                      "dumb-init --single-…"   8 weeks ago         Up 8 weeks                              kibana
7d22d71cc31b        kolla/centos-source-keepalived:rocky                  "dumb-init --single-…"   8 weeks ago         Up 8 weeks                              keepalived
dae774ca7e33        kolla/centos-source-haproxy:rocky                     "dumb-init --single-…"   8 weeks ago         Up 8 weeks                              haproxy
14b340bb8139        kolla/centos-source-redis-sentinel:rocky              "dumb-init --single-…"   8 weeks ago         Up 8 weeks                              redis_sentinel
3023e95f465f        kolla/centos-source-redis:rocky                       "dumb-init --single-…"   8 weeks ago         Up 8 weeks                              redis
a3ed7e8fe8ff        kolla/centos-source-elasticsearch:rocky               "dumb-init --single-…"   8 weeks ago         Up 8 weeks                              elasticsearch
06b28cd0f7c7        kolla/centos-source-zookeeper:rocky                   "dumb-init --single-…"   8 weeks ago         Up 8 weeks                              zookeeper
630219f5fb29        kolla/centos-source-chrony:rocky                      "dumb-init --single-…"   8 weeks ago         Up 8 weeks                              chrony
6f6189a4dfda        kolla/centos-source-cron:rocky                        "dumb-init --single-…"   8 weeks ago         Up 8 weeks                              cron
039f08ec1bbf        kolla/centos-source-kolla-toolbox:rocky               "dumb-init --single-…"   8 weeks ago         Up 8 weeks                              kolla_toolbox
f839d23859cc        kolla/centos-source-fluentd:rocky                     "dumb-init --single-…"   8 weeks ago         Up 8 weeks                              fluentd
[root@wuhan31-ceph01 ~]# 
```

四，常见故障

mariadb，这种情况是所有节点关机后遇到的。

vip 3306监听状态，节点3306非监听状态，容器反复重启中

可能原因是节点同时断开后，mariadb服务不可用，需要恢复服务

kolla-ansible -i kolla-ansible/inventory-xiaoxuantest mariadb\_recovery

执行完后确认各节点3306为监听状



遇到的问题：

ironic : Checking ironic-agent files exist for Ironic Inspector

```
TASK [ironic : Checking ironic-agent files exist for Ironic Inspector] ********************
failed: [localhost -> localhost] (item=ironic-agent.kernel) => {"changed": false, "failed_when_result": true, "item": "ironic-agent.kernel", "stat": {"exists": false}}
failed: [localhost -> localhost] (item=ironic-agent.initramfs) => {"changed": false, "failed_when_result": true, "item": "ironic-agent.initramfs", "stat": {"exists": false}} 
```

临时关闭enable\_ironic开头的配置解决.



neutron : Checking if ‘MountFlags’ for docker service is set to ‘shared’

```
TASK [neutron : Checking if 'MountFlags' for docker service is set to 'shared'] ***********
fatal: [localhost]: FAILED! => {"changed": false, "cmd": ["systemctl", "show", "docker"], "delta": "0:00:00.010391", "end": "2018-12-24 20:44:46.791156", "failed_when_result": true, "rc": 0, "start": "2018-12-24 20:44:46.780765",...
```

见章节 环境准备–docker–



ceilometer : Checking gnocchi backend for ceilometer

```
TASK [ceilometer : Checking gnocchi backend for ceilometer] *******************************
fatal: [localhost -> localhost]: FAILED! => {"changed": false, "msg": "gnocchi is required but not enabled"}
```



启用 gnocchi

octavia : Checking certificate files exist for octavia

```
TASK [octavia : Checking certificate files exist for octavia] *****************************
failed: [localhost -> localhost] (item=cakey.pem) => {"changed": false, "failed_when_result": true, "item": "cakey.pem", "stat": {"exists": false}}
failed: [localhost -> localhost] (item=ca_01.pem) => {"changed": false, "failed_when_result": true, "item": "ca_01.pem", "stat": {"exists": false}}
failed: [localhost -> localhost] (item=client.pem) => {"changed": false, "failed_when_result": true, "item": "client.pem", "stat": {"exists": false}}
```

运行 kolla-ansible certificates 依旧没有生成, 查了下, 官方还没修复:  https://bugs.launchpad.net/kolla-ansible/+bug/1668377

先禁用 octavia . 后期排查了人工生成.



common : Restart fluentd container

```
RUNNING HANDLER [common : Restart fluentd container] **************************************
fatal: [localhost]: FAILED! => {"changed": false, "msg": "Unknown error message: Get https://192.168.55.201:4000/v1/_ping: dial tcp 100.100.31.201:4000: getsockopt: connection refused"}
```

看了下, 确实没启动4000端口. 根据官方文档\[参1\]部署了registry.

