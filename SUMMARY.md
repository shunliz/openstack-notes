# Summary

* [前言](README.md)
* [概述](introduction.md)
* [第一部 计算](compute.md)
  * [计算硬件](compute/lqk/ji-suan-ying-jian.md)
    * [CPU](compute/lqk/ji-suan-ying-jian/cpu.md)
    * [GPU](compute/lqk/ji-suan-ying-jian/gpu.md)
    * [DPU](compute/lqk/ji-suan-ying-jian/dpu.md)
    * [FPGA](compute/lqk/ji-suan-ying-jian/fpga.md)
  * [体系结构](compute/lqk/ti-xi-jie-gou.md)
    * SMP
    * MIMD
  * [Libvirt+Qemu+KVM](compute/lqk/lqk.md)
    * [vCPU&pCPU](compute/lqk/lqk/vcpuandpcpu.md)
    * [内存虚拟化](compute/lqk/lqk/nei-cun-xu-ni-hua.md)
    * [Qemu](compute/lqk/lqk/qemu.md)
      * [TCG](compute/lqk/lqk/tcg.md)
    * qemu kvm运行过程
  * [Nova](compute/nova/nova.md)
    * [虚拟机初始配置（key,password,script）](compute/nova/nova/xu-ni-ji-chu-shi-pei-zhi-ff08-key-password-script.md)
    * [热迁移](compute/nova/nova/re-qian-yi.md)
    * [Cloud-init](compute/nova/nova/cloud-init.md)
    * [libguestfs-tools](compute/nova/nova/libguestfs-tools.md)
  * [Baremetal](compute/baremetal/baremetal.md)
  * [容器](compute/container/container.md)
    * [Docker](compute/container/container/docker.md)
    * [K8S](compute/container/container/k8s.md)
      * [K8S基础](compute/container/container/k8sji-chu.md)
      * [kube-ovn](compute/container/container/kube-ovn.md)
      * [virtual-kubelet](compute/container/container/virtual-kubelet.md)
      * [kubervirt](compute/container/container/kubervirt.md)
      * [Cluster-API](compute/container/container/cluster-api.md)
      * [Kubesphere](compute/container/container/kubesphere.md)
      * [helm&tiller](compute/container/container/helmandtiller.md)
      * cilium
      * gpu
      * [istio](compute/container/container/istio.md)
        * [请求路由](compute/container/container/istio/qing-qiu-lu-you.md)
        * [错误注入](compute/container/container/istio/cuo-wu-zhu-ru.md)
        * [流量切换](compute/container/container/istio/liu-liang-qie-huan.md)
        * 查询指标
        * [部署模型](compute/container/container/istio/bu-shu-mo-xing.md)
      * linkerd
      * inaccel
      * knative
      * [kubeflow](compute/container/container/kubeflow.md)
      * multus
      * openebs
      * openfaas
      * [minikube](compute/container/container/minikube.md)
      * [microk8s](compute/container/container/microk8s.md)
      * [Kubevela](compute/container/container/kubevela.md)
      * [OpenKruise](compute/container/container/openkruise.md)
    * [K8S开发](compute/container/container/k8skai-fa.md)
      * client-go
      * kubebuilder
* [第二部 网络](network.md)
  * [网络基础理论](network/basic/basic.md)
    * [TCP/IP网络模型](network/physicalnet/physicalnet/tcpipwang-luo-mo-xing.md)
    * [ARP](network/physicalnet/physicalnet/arp.md)
    * [Overlay网络](network/physicalnet/physicalnet/overlaywang-luo.md)
    * [交换](network/physicalnet/physicalnet/jiao-huan.md)
      * STP
      * [VLAN](network/physicalnet/physicalnet/jiao-huan/vlan.md)
    * [路由](network/physicalnet/physicalnet/lu-you.md)
      * [BGP](network/physicalnet/physicalnet/lu-you/bgp.md)
      * [OSPF](network/physicalnet/physicalnet/lu-you/ospf.md)
    * [TCP](network/physicalnet/physicalnet/tcp.md)
    * [VRRP](network/physicalnet/physicalnet/vrrp.md)
  * [物理网络](network/physicalnet/physicalnet.md)
    * [5G](network/physicalnet/physicalnet/5g.md)
  * [虚拟网络](network/vnetwork/vnetwork.md)
    * [Linux网络](network/vnetwork/linuxwang-luo.md)
      * [Linux网络配置](network/vnetwork/linuxwang-luo-pei-zhi.md)
      * [虚拟网络设备Tap/Tun/veth](network/vnetwork/xu-ni-wang-luo-she-bei.md)
      * [虚拟网络设备网桥](network/vnetwork/xu-ni-wang-luo-she-bei-wang-qiao.md)
      * [linux路由](network/vnetwork/linuxlu-you.md)
      * [内核VRF](network/vnetwork/linuxwang-luo/nei-he-vrf.md)
      * [数据发送过程](network/vnetwork/shu-ju-fa-song-guo-cheng.md)
      * [数据包的接收过程](network/vnetwork/shu-ju-bao-de-jie-shou-guo-cheng.md)
      * [负载均衡](network/vnetwork/fu-zai-jun-heng.md)
      * [eBPF](network/vnetwork/ebpf.md)
        * [bcc](network/vnetwork/ebpf/bcc.md)
        * [故障排查](network/vnetwork/ebpf/gu-zhang-pai-cha.md)
      * [XDP](network/vnetwork/xdp.md)
      * [iptables/netfilter](network/vnetwork/iptablesnetfilter.md)
      * [SR-IOV](network/vnetwork/sr-iov.md)
      * 常用工具
        * [tcpdump](network/vnetwork/tcpdump.md)
        * [scapy](network/vnetwork/scapy.md)
    * [OVS](network/vnetwork/ovs.md)
      * [OVS原理](network/vnetwork/ovs/ovsyuan-li.md)
      * [OVS编译](network/vnetwork/ovs/ovsbian-yi.md)
      * [OVN-Openstack](network/vnetwork/ovs/ovn-openstack.md)
      * [VXLAN](network/vnetwork/ovs/vxlan.md)
    * [OVN](network/vnetwork/ovn.md)
      * [OVN和流表](network/vnetwork/ovn-flowtable.md)
        * [OVN分功能流表](network/vnetwork/ovn-flowtable/ovnfen-gong-neng-liu-biao.md)
      * [OVN编译](network/vnetwork/ovnbian-yi.md)
      * [OVN高可用](network/vnetwork/ovngao-ke-yong.md)
      * OVN-K8S
      * OVN-Docker
      * [OVN-Openstack](network/vnetwork/ovn-openstack.md)
    * [容器网络](network/vnetwork/rong-qi-wang-luo.md)
      * [host network](network/vnetwork/host-network.md)
      * [k8s网络](network/vnetwork/k8swang-luo.md)
      * CNI
      * CNM
    * [流量控制](network/vnetwork/liu-liang-kong-zhi.md)
      * [TCP BBR](network/vnetwork/tcp-bbr.md)
    * DPDK
      * VPP
  * [neutron](network/neutron/neutron.md)
    * [Neutron OVN网络流量分析](network/vnetwork/neutron-ovn.md)
    * [ML2&Vxlan](network/vnetwork/ml2andvxlan.md)
    * [L2Gateway](network/vnetwork/l2gateway.md)
    * [OpenStack Neutron之层次化端口绑定](network/vnetwork/openstack-neutronzhi-ceng-ci-hua-duan-kou-bang-ding.md)
    * [Octavia](network/vnetwork/octavia.md)
    * Kuryr Kubernetes
    * [PortBinding](network/vnetwork/portbinding.md)
* [第三部 存储](storage.md)
  * [物理存储](storage/phystorage/phystorage.md)
    * 存储协议
      * [SCSI](storage/phystorage/phystorage/scsi.md)
      * [iSCSI](storage/phystorage/phystorage/iscsi.md)
      * FC
      * SAS
      * [NVMe-oF](storage/phystorage/phystorage/nvme-of.md)
      * RDMA
  * 虚拟存储
    * virtio-blk
    * [LVM](storage/phystorage/lvm.md)
    * [SPDK](storage/phystorage/spdk.md)
      * NVMe
      * RDMA
  * [分布式存储](storage/distributed/distributed.md)
  * [Cinder](storage/cinder/cinder.md)
    * [Cinder卷的加密](storage/cinder/cinder/cinderjuan-de-jia-mi.md)
    * [cinder-lvm](storage/cinder/cinder/cinder-lvm.md)
  * [Ceph](storage/ceph/ceph.md)
    * rbd
    * cephfs
    * radosgw
    * [ceph rbd灾备方案](storage/ceph/ceph/cephzai-bei-fang-an.md)
      * ceph rbd mirror
    * [crush](storage/ceph/ceph/crush.md)
    * [ceph调优](storage/ceph/ceph/cephdiao-you.md)
    * [ceph-cache-tier](storage/ceph/ceph/ceph-cache-tier.md)
    * [Ceph集群中的逻辑结构](storage/ceph/ceph/cephji-qun-zhong-de-luo-ji-jie-gou.md)
* [第四部 其它](ops.md)
  * [安装部署](ops/install/install.md)
    * [OpenStack Ansible](ops/install/install/openstack-ansible.md)
    * [Kolla Ansible](ops/install/install/kolla-ansible.md)
    * [Kolla Ansible2](ops/install/install/kolla-ansible2.md)
    * [packer](ops/install/install/packer.md)
    * [terraform](ops/install/install/terraform.md)
    * [Ansible](ops/install/install/ansible.md)
    * [IAC基础设施即代码](ops/install/install/iacji-chu-she-shi-ji-dai-ma.md)
  * [监控](ops/monitor/monitor.md)
    * [云合规监控](ops/monitor/monitor/yun-he-gui-jian-kong.md)
  * [运维](ops/ops/ops.md)
  * [安全](ops/install/an-quan.md)

