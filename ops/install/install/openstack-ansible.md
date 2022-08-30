Openstack Ansible部署OVN raft cluster

[https://satishdotpatel.github.io/openstack-ansible-inventory/](https://satishdotpatel.github.io/openstack-ansible-inventory/)

[https://satishdotpatel.github.io/openstack-ansible-ovn-deployment-part1/](https://satishdotpatel.github.io/openstack-ansible-ovn-deployment-part1/)

[https://satishdotpatel.github.io/openstack-ansible-ovn-deployment-part2/](https://satishdotpatel.github.io/openstack-ansible-ovn-deployment-part2/)

[https://satishdotpatel.github.io/openstack-ansible-ovn-clustering/](https://satishdotpatel.github.io/openstack-ansible-ovn-clustering/)

centos\_mirror_url：proxy\_url_

\# openstack-ansible setup-hosts.yml

\# openstack-ansible setup-infrastructure.yml

\# openstack-ansible setup-openstack.yml

第三步未测试

网络规划

```
Network                    CIDR             VLAN      bond        bridge
Management Network     10.0.244.0/24     240      bond0         br-mgmt
Tunnel (VXLAN) Network 172.29.240.0/24     999      bond1         br-vxlan
Storage Network           172.29.244.0/24   888      bond0         br-storage

IP assignments
The following host name and IP address assignments are used for this environment.
Host name          Management IP    Tunnel (VxLAN) IP    Storage IP
lb_vip_address     10.0.244.1          
infra1           10.0.244.6       172.29.240.11     172.29.244.11
infra2           10.0.244.7       172.29.240.12     172.29.244.12
infra3           10.0.244.8       172.29.240.13     172.29.244.13
log1 + NFS Storage 10.0.244.9                          172.29.244.15                                                          
compute1       10.0.244.21       172.29.240.16     172.29.244.16
compute2       10.0.244.22       172.29.240.17     172.29.244.17
deploy           10.0.240.115
```

基础环境配置

```
1、升级系统包和内核
# yum upgrade

2、重启服务器
reboot

3、如果在操作系统安装期间未安装其他软件包，请安装
# yum install https://rdoproject.org/repos/openstack-rocky/rdo-release-rocky.rpm
# yum install git ntp ntpdate openssh-server python-devel sudo '@Development Tools'

4、配置NTP以与适当的时间源同步
我自搭ntp、dns、以及repo在虚拟化环境以及提前搭好

5、默认情况下，在大多数CentOS系统上启用Firewalld服务，其默认规则集阻止OpenStack组件正常通信。停止firewalld服务并屏蔽它以防止它启动：
# systemctl stop firewalld
# systemctl mask firewalld

6、使用国内pip源
cat > /etc/pip.conf <<     EOF 
[global]
index-url = https://pypi.doubanio.com/simple
EOF

7、由于github在新加坡amazon云中，在部署时候发现经常失败，通过测试发现是路由环路，所以我绕过了新加坡，直接从美国github上来安装
cat > /etc/hosts  <<     EOF
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.30.255.112 github.com
EOF
```

部署机安装

```
8、配置网络默认使用manager网络
br-mgmt:
Container management: 172.29.236.0/22 (VLAN 10)

9、下载ansible文件
# git clone -b 18.1.5 https://git.openstack.org/openstack/openstack-ansible /opt/openstack-ansible

# # List all existing tags.
# git tag -l

# # Checkout the stable branch and find just the latest tag
# git checkout stable/rocky
# git describe --abbrev=0 --tags

# # Checkout the latest tag from either method of retrieving the tag.
# git checkout 18.1.5

10、更改为/opt/openstack ansible目录，并运行ansible引导脚本：
# cd /opt/openstack-ansible/
# scripts/bootstrap-ansible.sh

PLAY RECAP *******************************************************************************************************************************************************************************************************************************************************************
localhost                  : ok=4    changed=3    unreachable=0    failed=0   
+ popd
/opt/openstack-ansible
+ unset ANSIBLE_LIBRARY
+ unset ANSIBLE_LOOKUP_PLUGINS
+ unset ANSIBLE_FILTER_PLUGINS
+ unset ANSIBLE_ACTION_PLUGINS
+ unset ANSIBLE_CALLBACK_PLUGINS
+ unset ANSIBLE_CALLBACK_WHITELIST
+ unset ANSIBLE_TEST_PLUGINS
+ unset ANSIBLE_VARS_PLUGINS
+ unset ANSIBLE_STRATEGY_PLUGINS
+ unset ANSIBLE_CONFIG
+ '[' false == true ']'
+ echo 'System is bootstrapped and ready for use.'
System is bootstrapped and ready for use.
```

配置deploy-openstack配置文件

```
11、准备配置文件
# cp -av  /opt/openstack-ansible/etc/openstack_deploy /etc/

12、编辑openstack配置文件
# cd /etc/openstack_deploy
# cp openstack_user_config.yml.example  /etc/openstack_deploy/openstack_user_config.yml -av

13、配置让所有容器都使用国内pip源
vim /etc/openstack_deploy/user_variables.yml 
install_method: source
# Copy these files from the host into the containers
lxc_container_cache_files_from_host:
  - /etc/pip.conf

14、生成密钥
# cd /opt/openstack-ansible
# ./scripts/pw-token-gen.py --file /etc/openstack_deploy/user_secrets.yml
Creating backup file [ /etc/openstack_deploy/user_secrets.yml.tar ]
Operation Complete, [ /etc/openstack_deploy/user_secrets.yml ] is ready

15、检查部署服务器密钥是否生成
# ll /root/.ssh/
total 16
-rw-r--r-- 1 root root  396 Apr  3 19:45 authorized_keys
-rw------- 1 root root 1679 Apr  3 19:45 id_rsa
-rw-r--r-- 1 root root  396 Apr  3 19:45 id_rsa.pub
-rw-r--r-- 1 root root  171 Apr  4 08:56 known_hosts

16、检查配置文件
# openstack-ansible setup-infrastructure.yml --syntax-check
playbook: setup-infrastructure.yml

EXIT NOTICE [Playbook execution success] **************************************
===============================================================================

17、启动部署
# openstack-ansible setup-hosts.yml


18、配置文件：
# cat /etc/openstack_deploy/openstack_user_config.yml
---
cidr_networks:
  container: 10.0.244.0/24
  tunnel: 172.29.240.0/22
  storage: 172.29.244.0/22

used_ips:
  - "10.0.244.1,10.0.244.50"
  - "10.0.244.100"

global_overrides:
  internal_lb_vip_address: 10.0.244.1
  external_lb_vip_address: "openstack.shuyuan.com"
  management_bridge: "br-mgmt"
  provider_networks:
    - network:
        container_bridge: "br-mgmt"
        container_type: "veth"
        container_interface: "eth1"
        ip_from_q: "container"
        type: "raw"
        group_binds:
          - all_containers
          - hosts
        is_container_address: true
    - network:
        container_bridge: "br-vxlan"
        container_type: "veth"
        container_interface: "eth10"
        ip_from_q: "tunnel"
        type: "vxlan"
        range: "1:1000"
        net_name: "vxlan"
        group_binds:
          - neutron_linuxbridge_agent
    - network:
        container_bridge: "br-vlan"
        container_type: "veth"
        container_interface: "eth12"
        host_bind_override: "eth12"
        type: "flat"
        net_name: "flat"
        group_binds:
          - neutron_linuxbridge_agent
    - network:
        container_bridge: "br-storage"
        container_type: "veth"
        container_interface: "eth2"
        ip_from_q: "storage"
        type: "raw"
        group_binds:
          - glance_api
          - cinder_api
          - cinder_volume
          - nova_compute

# galera, memcache, rabbitmq, utility
shared-infra_hosts:
  infra1:
    ip: 10.0.244.6
    container_vars:
      container_tech: lxc
  infra2:
    ip: 10.0.244.7
  infra3:
    ip: 10.0.244.8

# repository (apt cache, python packages, etc)
repo-infra_hosts:
  infra1:
    ip: 10.0.244.6
  infra2:
    ip: 10.0.244.7
  infra3:
    ip: 10.0.244.8

# load balancer
# Ideally the load balancer should not use the Infrastructure hosts.
# Dedicated hardware is best for improved performance and security.
haproxy_hosts:
  infra1:
    ip: 10.0.244.6
  infra2:
    ip: 10.0.244.7
  infra3:
    ip: 10.0.244.8
# rsyslog server
log_hosts:
  log1:
    ip: 10.0.244.9

###
### OpenStack
###

# keystone
identity_hosts:
  infra1:
    ip: 10.0.244.6
  infra2:
    ip: 10.0.244.7
  infra3:
    ip: 10.0.244.8

# cinder api services
storage-infra_hosts:
  infra1:
    ip: 10.0.244.6
  infra2:
    ip: 10.0.244.7
  infra3:
    ip: 10.0.244.8

# glance
# The settings here are repeated for each infra host.
# They could instead be applied as global settings in
# user_variables, but are left here to illustrate that
# each container could have different storage targets.
image_hosts:
  infra1:
    ip: 10.0.244.6
    container_vars:
      limit_container_types: glance
      glance_nfs_client:
        - server: "172.29.244.15"
          remote_path: "/images"
          local_path: "/var/lib/glance/images"
          type: "nfs"
          options: "_netdev,auto"
  infra2:
    ip: 10.0.244.7
    container_vars:
      limit_container_types: glance
      glance_nfs_client:
        - server: "172.29.244.15"
          remote_path: "/images"
          local_path: "/var/lib/glance/images"
          type: "nfs"
          options: "_netdev,auto"
  infra3:
    ip: 10.0.244.8
    container_vars:
      limit_container_types: glance
      glance_nfs_client:
        - server: "172.29.244.15"
          remote_path: "/images"
          local_path: "/var/lib/glance/images"
          type: "nfs"
          options: "_netdev,auto"

# nova api, conductor, etc services
compute-infra_hosts:
  infra1:
    ip: 10.0.244.6
  infra2:
    ip: 10.0.244.7
  infra3:
    ip: 10.0.244.8

# heat
orchestration_hosts:
  infra1:
    ip: 10.0.244.6
  infra2:
    ip: 10.0.244.7
  infra3:
    ip: 10.0.244.8

# horizon
dashboard_hosts:
  infra1:
    ip: 10.0.244.6
  infra2:
    ip: 10.0.244.7
  infra3:
    ip: 10.0.244.8

# neutron server, agents (L3, etc)
network_hosts:
  infra1:
    ip: 10.0.244.6
  infra2:
    ip: 10.0.244.7
  infra3:
    ip: 10.0.244.8

# ceilometer (telemetry data collection)
metering-infra_hosts:
  infra1:
    ip: 10.0.244.6
  infra2:
    ip: 10.0.244.7
  infra3:
    ip: 10.0.244.8

# aodh (telemetry alarm service)
metering-alarm_hosts:
  infra1:
    ip: 10.0.244.6
  infra2:
    ip: 10.0.244.7
  infra3:
    ip: 10.0.244.8

# gnocchi (telemetry metrics storage)
metrics_hosts:
  infra1:
    ip: 10.0.244.6
  infra2:
    ip: 10.0.244.7
  infra3:
    ip: 10.0.244.8

# nova hypervisors
compute_hosts:
  compute1:
    ip: 10.0.244.21
  compute2:
    ip: 10.0.244.22

# ceilometer compute agent (telemetry data collection)
metering-compute_hosts:
  compute1:
    ip: 10.0.244.21
  compute2:
    ip: 10.0.244.22

# cinder volume hosts (NFS-backed)
# The settings here are repeated for each infra host.
# They could instead be applied as global settings in
# user_variables, but are left here to illustrate that
# each container could have different storage targets.
storage_hosts:
  infra1:
    ip: 10.0.244.6
    container_vars:
      cinder_backends:
        limit_container_types: cinder_volume
        nfs_volume:
          volume_backend_name: NFS_VOLUME1
          volume_driver: cinder.volume.drivers.nfs.NfsDriver
          nfs_mount_options: "rsize=65535,wsize=65535,timeo=1200,actimeo=120"
          nfs_shares_config: /etc/cinder/nfs_shares
          shares:
            - ip: "172.29.244.15"
              share: "/vol/cinder"
  infra2:
    ip: 10.0.244.7
    container_vars:
      cinder_backends:
        limit_container_types: cinder_volume
        nfs_volume:
          volume_backend_name: NFS_VOLUME1
          volume_driver: cinder.volume.drivers.nfs.NfsDriver
          nfs_mount_options: "rsize=65535,wsize=65535,timeo=1200,actimeo=120"
          nfs_shares_config: /etc/cinder/nfs_shares
          shares:
            - ip: "172.29.244.15"
              share: "/vol/cinder"
  infra3:
    ip: 10.0.244.8
    container_vars:
      cinder_backends:
        limit_container_types: cinder_volume
        nfs_volume:
          volume_backend_name: NFS_VOLUME1
          volume_driver: cinder.volume.drivers.nfs.NfsDriver
          nfs_mount_options: "rsize=65535,wsize=65535,timeo=1200,actimeo=120"
          nfs_shares_config: /etc/cinder/nfs_shares
          shares:
            - ip: "172.29.244.15"
              share: "/vol/cinder"
```

/etc/openstack\_deploy/user\_variables.yml

```
---
debug: false
apply_security_hardening: false
install_method: source
neutron_plugin_type: ml2.ovn
neutron_plugin_base:
  - neutron.services.ovn_l3.plugin.OVNL3RouterPlugin
neutron_ml2_drivers_type: "vlan,local,geneve"
```

/etc/openstack\_deploy/env.d/neutron.yml

```
component_skel:
  neutron_ovn_controller:
    belongs_to:
      - neutron_all
  neutron_ovn_northd:
    belongs_to:
      - neutron_all

container_skel:
  neutron_agents_container:
    contains: {}
  neutron_ovn_northd_container:
    belongs_to:
      - network_containers
    contains:
      - neutron_ovn_northd
```

/etc/openstack\_deploy/env.d/nova.yml

```
container_skel:
  nova_compute_container:
    belongs_to:
      - compute_containers
      - kvm-compute_containers
      - lxd-compute_containers
      - qemu-compute_containers
    contains:
      - neutron_ovn_controller
      - nova_compute
    properties:
      is_metal: true
```

/etc/openstack\_deploy/group\_vars/network\_hosts

```
openstack_host_specific_kernel_modules:
  - name: "openvswitch"
    pattern: "CONFIG_OPENVSWITCH"
```



