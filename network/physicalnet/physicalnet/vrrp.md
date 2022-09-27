# 什么是VRRP?

> 虚拟路由冗余协议VRRP（Virtual Router Redundancy Protocol）是一种用于提高网络可靠性的容错协议。通过VRRP，可以在主机的下一跳设备出现故障时，及时将业务切换到备份设备，从而保障网络通信的连续性和可靠性。

# 为什么需要VRRP?

⭐ 随着网络的快速普及和相关应用的日益深入，各种增值业务（如IPTV、视频会议等）已经开始广泛部署，基础网络的可靠性日益成为用户关注的焦点，能够保证网络传输不中断对于终端用户非常重要。

⭐ 现网中的主机使用缺省网关与外部网络联系时，如果Gateway出现故障，与其相连的主机将与外界失去联系，导致业务中断。

⭐ VRRP的出现很好地解决了这个问题。VRRP将多台设备组成一个虚拟设备，通过配置虚拟设备的IP地址为缺省网关，实现缺省网关的备份。当网关设备发生故障时，VRRP机制能够选举新的网关设备承担数据流量，从而保障网络的可靠通信。如下图所示，当Master设备故障时，发往缺省网关的流量将由Backup设备进行转发。

# VRRP工作原理

## 1、VRRP的三种状态

VRRP协议中定义了三种状态机：初始状态（Initialize）、活动状态（Master）、备份状态（Backup）。其中，只有处于Master状态的设备才可以转发那些发送到虚拟IP地址的报文。

⭐ Initialize： 该状态为VRRP不可用状态，在此状态时设备不会对VRRP通告报文做任何处理。

通常设备启动时或设备检测到故障时会进入Initialize状态。

⭐ Master当VRRP设备处于Master状态时，它将会承担虚拟路由设备的所有转发工作，并定期向整个虚拟内发送VRRP通告报文。

⭐ Backup：当VRRP设备处于Backup状态时，它不会承担虚拟路由设备的转发工作，并定期接受Master设备的VRRP通告报文，判断Master的工作状态是否正常。

## 2、VRRP的选举机制

由几台路由器组成的虚拟路由器又称为VRRP备份组。一个VRRP备份组在逻辑上为一台路由器。VRRP备份组建立后，各设备会根据所配置的优先级来选举Master设备，选举方式如下图所示。  
![](/assets/network-netbasic-vrrp1.png)

## 3、VRRP工作原理

![](/assets/network-netbasic-vrrp2.png)

当Master设备出现故障时，路由器B和路由器C会选举出新的Master设备。新的Master设备开始响应对虚拟IP地址的ARP响应，并定期发送VRRP通告报文。

VRRP的详细工作过程如下：

⭐ VRRP备份组中的设备根据优先级选举出Master。Master设备通过发送免费ARP报文，将虚拟MAC地址通知给与它连接的设备或者主机，从而承担报文转发任务。

⭐ Master设备周期性向备份组内所有Backup设备发送VRRP通告报文，通告其配置信息（优先级等）和工作状况。

⭐ 如果Master设备出现故障，VRRP备份组中的Backup设备将根据优先级重新选举新的Master。 @

VRRP备份组状态切换时，Master设备由一台设备切换为另外一台设备，新的Master设备会立即发送携带虚拟路由器的虚拟MAC地址和虚拟IP地址信息的免费ARP报文，刷新与它连接的设备或者主机的MAC表项，从而把用户流量引到新的Master设备上来，整个过程对用户完全透明。

⭐ 原Master设备故障恢复时，若该设备为IP地址拥有者（优先级为255），将直接切换至Master状态。若该设备优先级小于255，将首先切换至Backup状态，且其优先级恢复为故障前配置的优先级。

⭐ Backup设备的优先级高于Master设备时，由Backup设备的工作方式（抢占方式和非抢占方式）决定是否重新选举Master。

## keepalived配置解释

```
! Configuration File for keepalived
global_defs {
  script_user root
  enable_script_security
}
vrrp_script check_nginx {
    script "/etc/keepalived/nginx_check.sh"
    interval 10
}
vrrp_instance VI_1 {  # 定义一个实例
    state BACKUP     # 指定Keepalived的角色，MASTER表示此主机是主服务器,BACKUP表示此主机是备用服务器，所以设置priority时要注意MASTER比BACKUP高。如果设置了nopreempt,那么state的这个值不起作用，主备靠priority决定。
    nopreempt    # 设置为不抢占 
    interface eth0   #指定监测网络的接口，当LVS接管时，将会把IP地址添加到该网卡上。
    virtual_router_id 101      #虚拟路由标识，同一个vrrp实例使用唯一的标识，同一个vrrp_instance下，MASTER和BACKUP必须一致。
    priority 100       #指定这个实例优先级
    unicast_src_ip 192.168.1.14  # 配置单播的源地址
    unicast_peer { 
        192.168.1.15       #配置单播的目标地址
    }    #keepalived在组播模式下所有的信息都会向224.0.0.18的组播地址发送，产生众多的无用信息，并且会产生干扰和冲突，可以将组播的模式改为单拨。这是一种安全的方法，避免局域网内有大量的keepalived造成虚拟路由id的冲突。
    advert_int 1      #心跳报文发送间隔
    authentication {
        auth_type PASS    #设置验证类型，主要有PASS和AH两种
        auth_pass test123   #设置验证密码，同一个vrrp_instance下，MASTER和BACKUP的密码必须一致才能正常通信
    }
    virtual_ipaddress {    #设置虚拟IP地址，可以设置多个虚拟IP地址，每行一个
        118.24.101.16/24 dev eth1 
    }
    track_interface {  # 设置额外的监控，里面那个网卡出现问题都会切换
        eth0
    }
    track_script {
        check_nginx
    }
}
```

问题：两台机器上面都有VIP的情况

排查：

1.检查防火墙，发现已经是关闭状态。

1. keepalived.conf配置问题。

3.可能是上联交换机禁用了arp的广播限制，造成keepalive无法通过广播通信，两台服务器抢占vip，出现同时都有vip的情况。

tcpdump -i eth0 vrrp -n 检查发现 14和15都在对224.0.0.18发送消息。但是在正常情况下，备节点如果收到主节点的心跳消息时，优先级高于自己，就不会主动对外发送消息。

解决方法，将多播调整为单播然后重启服务：

\[root@test-15\]\# vim /etc/keepalived.conf

```
priority 50

unicast\_src\_ip  172.19.1.15   \#本机ip

unicast\_peer {              

    172.19.1.14      \#对端ip

}
```

\[root@test-14\]\# vim /etc/keepalived.conf

```
priority 100

unicast\_src\_ip  172.19.1.14   \#本机ip

unicast\_peer {              

    172.19.1.15      \#对端ip

}
```

配置完成后恢复正常，查看：

** tcpdump -i eth0 vrrp -n**

```
16:38:45.085456 IP 192.168.1.14 > 192.168.1.15: VRRPv2, Advertisement, (ttl 254), vrid 101, prio 150, authtype simple, intvl 1s, length 20
16:38:45.097735 IP 192.168.1.125 > 224.0.0.18: VRRPv2, Advertisement, vrid 91, prio 101, authtype simple, intvl 1s, length 20
16:38:45.098797 IP 192.168.1.6 > 224.0.0.18: VRRPv2, Advertisement, vrid 60, prio 102, authtype simple, intvl 1s, length 24
16:38:45.098941 IP 192.168.1.59 > 224.0.0.18: VRRPv2, Advertisement, vrid 127, prio 150, authtype simple, intvl 1s, length 20
16:38:45.104014 IP 192.168.1.110 > 224.0.0.18: VRRPv2, Advertisement, vrid 171, prio 102, authtype simple, intvl 1s, length 20
16:38:46.086591 IP 192.168.1.14 > 192.168.1.15: VRRPv2, Advertisement, (ttl 254), vrid 101, prio 150, authtype simple, intvl 1s, length 20
16:38:46.098630 IP 192.168.1.125 > 224.0.0.18: VRRPv2, Advertisement, vrid 91, prio 101, authtype simple, intvl 1s, length 20
16:38:46.099057 IP 192.168.1.59 > 224.0.0.18: VRRPv2, Advertisement, vrid 127, prio 150, authtype simple, intvl 1s, length 20
16:38:46.104108 IP 192.168.1.110 > 224.0.0.18: VRRPv2, Advertisement, vrid 171, prio 102, authtype simple, intvl 1s, length 20
16:38:47.087652 IP 192.168.1.14 > 192.168.1.15: VRRPv2, Advertisement, (ttl 254), vrid 101, prio 150, authtype simple, intvl 1s, length 20
```



