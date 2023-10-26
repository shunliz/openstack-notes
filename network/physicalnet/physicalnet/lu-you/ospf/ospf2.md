OSPF是一种基于链路状态的路由协议，也是专为 IP 开发的路由协议，直接运行在 IP 层上面。它从设计上保证了无路由环路。除此之外，IS-IS也是很常见的链路状态协议。

**为什么会出现OSPF？**

作为目前主流的IGP协议，OSPF主要是为了解决RIP的三大问题而出现的，比如：收敛很慢、容易产生路由环路以及可延展性差等。

OSPF路由协议的应用面非常广，认可度也很高，毕竟的确是好用的。

像教育，金融，运营商，企业、医疗等行业，不论组网模型是复杂还是简单，也无论设备数量和路由条目有多少，OSPF都能很好的满足对应的需求。

**所以，在网络部署IGP协议的时候，很多网工都会优先考虑用OSPF组网。**

  


对于OSPF，很多网工朋友都是在工作里用到了就看一点基础内容，很难全面掌握。

今天想给你分享的就是OSPF的最全原理总结（概要版），**对你构建大体思路有奇效，希望能对同行们有所帮助。**

  


  


**01**

**OSPF路由协议概述**

  


**1. 内部网关协议和外部网关协议**

* 自治系统\(AS\)
* 内部网关协议\(IGP\) ：rip、ospf等
* 外部网关协议\(EGP\)：bgp等

  


**2. OSPF的工作过程**

* 邻居列表
* 链路状态数据库
* 路由表

  


  


![](https://pic4.zhimg.com/80/v2-e355f2823d88243956b68cb202cd1f83_720w.webp)

  


  


**02**

**OSPF的应用环境**

  


**1. 从几方面考虑OSPF的使用**

* 网络规模
* 网络拓扑
* 其他特殊要求
* 路由器自身要求

  


**2. OSPF的特点**

* 可适应大规模网络
* 路由变化收敛速度快
* 无路由环
* 支持变长子网掩码VLSM
* 支持区域划分
* 支持以组播地址发送协议报

  


  


![](https://pic1.zhimg.com/80/v2-487d308a340f5d47a028c54851848130_720w.webp)

  


  


**03**

**OSPF基本概念**

  


**1. OSPF区域**

* 为了适应大型的网络，OSPF在AS内划分多个区域
* 每个OSPF路由器只维护所在区域的完整链路状态信息

  


**（1）区域ID**

* 区域ID可以表示成一个十进制的数字
* 也可以表示成一个IP

  


**（2）骨干区域Area 0**

* 负责区域间路由信息传播

  


**（3）非骨干区域**

* 非晋干区域相互通信必须通过骨干区域

– 标准区域

– 末梢区域stub

– 完全末梢区域total stub

– 非纯末悄区域nssa

  


  


![](https://pic3.zhimg.com/80/v2-1d3165540a48787bde9f6b1336340fda_720w.webp)

  


  


![](https://pic3.zhimg.com/80/v2-cc9280c42c25d8908c2b99601ef2b09a_720w.webp)

  


  


**2. OSPF路由类型**

* 区域之间路由器: ABR
* 自制系统边界路由器:ASBR

  


  


![](https://pic2.zhimg.com/80/v2-37b3ade2221b08695ad74a5c999dd795_720w.webp)

  


  


**3. 生成OSPF多区域的原因**

* 改善网络的可扩展性
* 快速收敛

  


**4. Router ID**

* OSPF区域内唯一标识路由器的IP地址

  


**5. Router ID选取规则**

* 选取路由器loopback接口上数值最高的IP地址
* 如果没有loopback接口，在物理端口中选取IP地址最高的
* 也可以使用router-id命令指定Router ID
* DR和BDR的选举方法

**6. 选举DR和BDR**

  


**（1）自动选举DR和BDR**

* 网段上Router lID最大的路由器将被选举为DR，第二大的将被选举为BDR

  


**（2）手工选择DR和BDR**

* 优先级范围是0～255，数值越大，优先级越高，默认为1
* 如果优先级相同，则需要比较Router ID
* 如果路由器的优先级被设置为0，它将不参与DR和DBR的选举

  


**（3）DR和BDR的选举过程**

* 路由器的优先级可以影响一个选举过程，但是它不能强制更换已经存在的DR或BDR路由器

  


  


![](https://pic4.zhimg.com/80/v2-5c8fe3d201b6acd691e749b0b88ad183_720w.webp)

  


  


![](https://pic3.zhimg.com/80/v2-b7fcf64accca5d0899b7b646aaa0a9ae_720w.webp)

  


  


**7. OSPF的组播地址**

224.0.0.5

224.0.0.6

  


DRothers向DR/BDR发送DBD、LASR或者Lsu时目标地址是224.0.0.6\(AllDRouter\)﹔或者理解为:DR/BDR侦224.0.0.6

  


DR/BDR向DRothers发送更新的DBD、LSR或者Lsu时目标地址是224.0.0.5\(AllSPFRouter\)，或者理解为:DRothers侦听224.0.0.5

  


**8. 度量值**

* OSPF度量值 cost（开销）=10OM/BW（端口带宽\)– 最短路径是基于接口指定的代（cost路径成本）计算的
* R工P是跳数

**9. OSPF的数据包类型**

承载在lIP数据包内，使用协议号89

  


OSPF的包类型：

  


  


![](https://pic3.zhimg.com/80/v2-54729ff008c2e03f2b510bdef270d496_720w.webp)

  


  


**10. OSPF协议7种状态分析**

OSPF启动的第一个阶段是使用Hello报文建立双向通信的过程；OSPF启动的第二个阶段是建立完全邻接关系。

  


  


![](https://pic1.zhimg.com/80/v2-9c02860266993ddaa179ae4a3ad29a44_720w.webp)

  


  


![](https://pic4.zhimg.com/80/v2-c3f707f0b0b491c34ff58fe3d630f20f_720w.webp)

  


  


**11. OSPF协议6种LSA分析**

  


  


![](https://pic1.zhimg.com/80/v2-db886d350d4bc1a8d7cb701e02f41b98_720w.webp)

  


  


每一种区域中允许泛洪的LSA：

  


  


![](https://pic3.zhimg.com/80/v2-a8708fb0191128c3d34c148dff844aca_720w.webp)

  


  


**12. OSPF地址汇总的作用**

* 地址汇总也是通过减少泛洪的LSA数量节省资源
* 可以通过屏蔽一些网络不稳定的细节来节省资源
* 减少路由表中的路由条目

  


  


  


**04**

**OSPF配置命令示例**

  


**1. 通用配置**

\[R1\]int g0/0/0 \#\#\#记置接口ip地址

\[R1-GigabitEthernet0/0/0\]ip add 11.0.0.2 24

\[Rl-GigabitEthernet0/0/o\]un sh

\[R1-GigabitEthernet0/0/0\]int g0/0/1

\[R1-GigabitEthernet0/0/1\]ip add 12.0.0.1 24

\[R1-GigabitEthernet0/0/1\]un sh

\[R1-GigabitEthernet0/0/1\]int lo o

\[R1-LoopBack0\]ip add 1.1.1.1 32

\[R1-LoopBack0\]ospf 1 router-id 1.1.1.1 \#\#\#创建OSPF进程，配置路由ID

\[R1-ospf-1\]area 1 \#\#\#进入区域1，区域ID可以用数字表示，也可以用IP表示，若区域o则是骨干区域

\[R1-ospf-1-area-0.0.0.1\]network 12.0.0.0 0.255.255.255 \#\# 宣告直连

\[R1-ospf-1-area-0.0.0.1\]network 1.1.1.1 0.0.0.0 \#\#宣告oSPF区域内的直连网段，使用反掩码

------------------------------------------------------------------------

reset ospf process \#\#\#重置oSPF进程

  


**2. 优化配置**

末梢区域和完全末梢区域的作用，其主要目的是减少区域内的LSa条目以及路由条目，减少对设备CPu和内存的占用；

  


末梢区域和完全末梢区域中ABR会自动生成一条默认路由发布到末梢区域或完全末梢区域中。

  


———–——–末梢区域配置命令（在ABR和区域内路由上配置）———–——–没有LSA4、5、7通告

  


\[R4\]ospf 1

\[R4-ospf-1\]area 2

\[R4-ospf-1\]network x.x.x.x x.x.x.x \#\#\#先宣告直连网段，再配优化

\[R4-ospf-1-area-0.0.0.2\]stub

\[R5\]display ip routing-table \#\#\#此时未梢区域中的路由会显示一条默认路由到外部区域

  


———–——–完全末梢区域配置命令（在ABR和区域内路由上配置）———–——–除一条LSA3的默认路由通告外，没有LSA3、4、5、7通告

  


\[R4\]ospf 1

\[R4-ospf-1\]area 2

\[R4-ospf-1\]network x.x.x.x x.x·x.x \#\#\#先宣告直连网段，再配优化

\[R4-ospf-1-area-0.0.0.2\]stub no-summary

\[R5\]display ip routing-table \#\#\#此时完全末梢区域中的路由会显示一条默认路由到除本区域外的其他区域

  


——————-完全非纯未梢区或配置命令{ABR和区域内路由（除ASBR）配置}———–——–没有LSA4、5通告

  


\[R4\]ospf 1

\[R4-ospf-1\]area 1

\[R4-ospf-1\]network x.x.x.x x.x.x.x \#先宣告直连网段，再配优化

\[R4-ospf-1-area-o.0.0.1\]nssa no-summary \#\#\#ABR配置

----------------------------------------------------------------------------

\[R4-ospf-1-area-o.o.o.1\]nssa \#\#\#域内路由配置

  


**3. 验证命令**

display ospf 1 peer brief \#\#\#查看本地设备上的OSPF 1的相关信息

display ospf 1 peer \#\#\#查看路由表中的OSPF路由（确定路由器的类型和属性）

display ospf 1 brief \#\#\#查看oSPF邻居表的简要信息

display ip routing-table \#\#\#查看oSPF邻居表的详细信息

display ospf routing

display ospf interface GigabitEthernet 0/0/o

  


**4. 查看LSA命令**

\[Huawei\]dis ospf lsdb router

\[Huawei\]dis ospf lsdb network

\[Huawei\]dis ospf lsdb summary

\[Huawei\]dis ospf lsdb asbr

\[Huawei\]dis ospf lsdb ase

\[Huawei\]dis ospf lsdb nssa

  


**5. 修改oSPF路由的接口优先集，缺省值为1**

\[R1\]int g0/0/0

\[Rl-GigabitEthernet0/0/0\]ospf dr-priority 1O

  


**6. OSPF路由重分发配置命令**

\[R1\]rip 1\#\#\#配置rip

\[Rl-rip-l\]version 2

\[Rl-rip-l\]undo summary

\[Rl-rip-1\]network 11.0.0.o

\[Rl-rip-1\]import-route ospf 1cost3 \#\#\#把ospf协议注入到rip进行路由重分发，路径类型缺省为路径类型2（外部开销），成本开销为3（对于rip的度量值是跳数\)，rip中重分发ospf要指定metric的值

\[Rl-rip-1\]ospf 1

\[R1-ospf-1\]import-route rip 1 type 1 cost 1 \#\#1把外部rip协议注入到oSPE进行路由重分发，使用路径类型1\(内部开销+外部开销），成本开销为1（COST=10OM/BW）

-------------------------------------------------------------------------------------------

\[Rl-ospf-1\]default-route-advertise always \#\#\#ospf重分发默认路由

\[R2-ospf-l\]import-route direct \#\#\#ospf重分发直连路由

\[R2-ospf-1\]import-route static \#\#\#ospf重分发静态路由

  


**7. 区域间路由汇总配置**

———–——–OSPF地址汇总计算示例———–——–

  


192.168.1.0/24—转换二进制 ——192.168.00000 001.0 /24

192.168.2.0/24—————————————192.168.00000 010.0/24

192.168.3.0/24—————————————192-168.00000 011.0/24

192.168.4.0/24—————————————192.168.00000 100.0/24

192.168.5.0/24—————————————192.168.00000 101.0/24

192.168.6.0/24—————————————192.168.00000 111.0/24

  


将二进制地址分成两部分（完全相同的前半部分和存在差异的后半部分），数出前半部分的位数（这里的192.168.00000为21位）

  


则汇总后的结果为：192.168.00000 000/21

———–——–区域间路由汇总配置（在ABR上配置）———–——–———–——–

  


\[R4\]ospf l

\[R4-ospf-l\]area 2

\[R4-ospf-1\]abr-summary 192.168.0.0 255.255.248.0

  


———–——–外部路由汇总配置（在ASBR上配置）———–——–———–——–

  


\[R5\]ospf l

\[R5-ospf-1\]area 2

\[R5-ospf-1\]asbr-summary 10.0.o.0 255.248.0.o

  


**8. 虚链路配置**

非骨干区域必须和骨干区域直接相连，若不与骨干区域直接相连，则需要在穿越一个非骨干区域的两台ABR之间配置虚链路。

  


虚拟链路的建立，是需要依靠底层的真实链路所在的区域来传输。

  


OSP:报文的\(hello等\)。所以如果底层的穿越传输区域不稳定的话，则导致上层的虚链路不稳定，影响整个网络的骨干区域的稳定性。

  


所以，一般不建议用这种法式。如果不得不使用，那么也仅仅是临时的解决方案。

  


———–——–在被穿越的非骨干区域的两揣ABR配置虚链路———–——–

  


-\[R2\]ospf 1

\[R2-ospf-1\]area 1

\[R2-ospf-l-area-o.o.0.1\]vlink-peer 1.1.1.1 \#\#\#相指定被穿越区域两端ABR的路由ID

  


----------------------------------------------------------------------------

  


\[Rl\]ospf 1

\[Rl-ospf-1\]area 1

\[R1-ospf-1-area-0.0.0.1\]vlink-peer 2.2.2.2 \#\#\#相指定被穿越区域两端ABR的路由ID

\[R1\]display ospf vlink \#\#\#查看本地上通过虚链路建立的oSPF邻居关系

