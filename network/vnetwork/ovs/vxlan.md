如果两个VM，连接到两个VTEP上，希望通过VXLAN相互通信，并且通信的时候，两个VM需要感知他们似乎在同一个VLAN下面。



![](/assets/network-virtualnet-ovs-vxlan1.png)



首先两个VTEP启动的时候，需要加入同一个组播组，通过IGMP协议。



![](/assets/network-virtualnet-ovs-vxlan2.png)

当VM1要发送包给VM2的时候，一开始VM1只知道VM2的IP地址，但是不知道VM2的MAC地址，如果他们在同一个VLAN里面，发现MAC地址这件事情，需要通过ARP协议，以广播的方式在VLAN里面询问，而是这个IP地址的机器VM2，会将MAC地址发送出来，于是VM1才能知道VM2的MAC地址。



既然按上面所述，VM1和VM2需要感觉上，他们是在同一个VLAN里面的，于是VM1就像原来一样发送了ARP请求。



ARP请求到达了VTEP1，VTEP1知道VM1不是在一个VLAN里面，而是在一个VXLAN里面，而且VM2也不归自己管，所以只好将ARP包通过VXLAN包头封装起来，然后通过组播，将ARP请求发送到所有的VTEP。



VTEP2知道VM2归自己管，于是将组播接收下啦，将VXLAN包头卸掉，将里面的ARP包发送给VM2，VM2看到ARP包，以为同一个VLAN里面的虚拟机在询问自己的MAC地址，于是将自己的MAC地址发送出来。



![](/assets/network-virtualnet-ovs-vxlan3.png)



VTEP2将ARP的结果封装VXLAN包头后发送给VTEP1，VTEP1将VXLAN包头卸掉后，将ARP的结果发送给VM1，VM1收到后感觉像从同一个VLAN里面发出来的一样。



在整个过程中，VTEP1和VTEP2都学习到了一条知识，就是VM1的MAC归VTEP1管，VM2的MAC归VTEP2管，以后两个VM通信的时候，不需要再组播问某个VM归谁管了，直接发送就可以了。





![](/assets/network-virtualnet-ovs-vxlan4.png)

当双方知道了MAC地址以后，两个VM之间的交互和GRE就很像了。



内部的IP和MAC地址被封装起来，外面加上VTEP的IP和MAC地址，发送到另一端后，再把外层卸下来，发送给对端的VM。



