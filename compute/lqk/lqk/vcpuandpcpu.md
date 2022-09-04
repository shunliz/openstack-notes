### [vCPU](https://ashiamd.github.io/docsify-notes/#/study/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E5%92%8C%E7%A1%AC%E4%BB%B6/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E7%B3%BB%E7%BB%9F%E5%9B%9E%E9%A1%BE?id=_141-vcpu) {#_141-vcpu}

* **一个Socket套接字对应一个CPU。**

* 然后一个CPU里面可以多核，每个核有可能超线程。

（假如一个CPU是4核8线程，那么每个CPU核超线程，一个核心能2个线程=&gt;实际就是2个ALU计算单元模拟双线程）

---

举例：

* 一个服务器4个CPU，每个CPU（4核4线程，`cpu cores:4`，`siblings:4`）， 那么这个CPU不具有超线程技术，总的vCPU即（4 \* 4 = 16）。

* 4个CPU，每个\(`cpu cores:4`，`siblings:8`\)，那么CPU超线程，总的vCPU即（4 \* 4 \* \(8/4\) \) = 32。

**vCPU即对应一个超线程（或线程）**。

---

#### [1.**物理CPU与VCPU的关系梳理**](https://ashiamd.github.io/docsify-notes/#/study/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E5%92%8C%E7%A1%AC%E4%BB%B6/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E7%B3%BB%E7%BB%9F%E5%9B%9E%E9%A1%BE?id=_1-%e7%89%a9%e7%90%86cpu%e4%b8%8evcpu%e7%9a%84%e5%85%b3%e7%b3%bb%e6%a2%b3%e7%90%86) {#_1-物理cpu与vcpu的关系梳理}

> [物理CPU与VCPU的关系梳理](https://support.huawei.com/enterprise/zh/knowledge/EKB1000080054)

##### [1.**系统可用的VCPU\*\***总数计算\*\*](https://ashiamd.github.io/docsify-notes/#/study/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E5%92%8C%E7%A1%AC%E4%BB%B6/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E7%B3%BB%E7%BB%9F%E5%9B%9E%E9%A1%BE?id=_1-%e7%b3%bb%e7%bb%9f%e5%8f%af%e7%94%a8%e7%9a%84vcpu%e6%80%bb%e6%95%b0%e8%ae%a1%e7%ae%97) {#_1-系统可用的vcpu总数计算}

* **系统可用的vCPU总数\(逻辑处理器\) = Socket数（CPU个数）x Core数（内核）x Thread数（超线程）**

* **1个VCPU = 1个超线程Thread**

如下图所示：

![](/assets/compute-libvirtkvmqemu-vcpuandpcpu.png)

* **在不做VCPU预留的条件下，系统可以创建或运行VM的VCPU总数可远远大于实际可提供的VCPU数目**
  。

##### [2. 虚拟机VCPU的分配与调度](https://ashiamd.github.io/docsify-notes/#/study/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E5%92%8C%E7%A1%AC%E4%BB%B6/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E7%B3%BB%E7%BB%9F%E5%9B%9E%E9%A1%BE?id=_2-%e8%99%9a%e6%8b%9f%e6%9c%bavcpu%e7%9a%84%e5%88%86%e9%85%8d%e4%b8%8e%e8%b0%83%e5%ba%a6) {#_2-虚拟机vcpu的分配与调度}

​**对虚拟机来说，不直接感知物理CPU，虚拟机的计算单元通过vCPU对象来呈现。虚拟机只看到VMM呈现给它的vCPU**。在VMM中，每个vCPU对应一个VMCS（Virtual-Machine Control Structure）结构，当VCPU被从物理CPU上切换下来的时候，其运行上下文会被保存在其对应的VMCS结构中；当VCPU被切换到PCPU上运行时，其运行上下文会从对应的VMCS结构中导入到物理CPU上。通过这种方式，实现各vCPU之间的独立运行。

​ 从虚拟机系统的结构与功能划分可以看出，客户操作系统与虚拟机监视器共同构成了虚拟机系统的两级调度框架，如图所示是一个多核环境下虚拟机系统的两级调度框架。**客户操作系统负责第2 级调度,即线程或进程在vCPU 上的调度（将核心线程映射到相应的VCPU上）**。虚拟机监视器负责第1 级调度, 即vCPU在物理处理单元上的调度。两级调度的调度策略和机制不存在依赖关系。vCPU调度器负责物理处理器资源在各个虚拟机之间的分配与调度,本质上即把各个虚拟机中的vCPU按照一定的策略和机制调度在物理处理单元上可以采用任意的策略来分配物理资源, 满足虚拟机的不同需求。vCPU可以调度在一个或多个物理处理单元执行（**分时复用或空间复用物理处理单元**）, 也可以与物理处理单元建立一对一固定的映射关系（限制访问指定的物理处理单元）。

![](/assets/compute-libvirtqemukvm-vcpuandpcpu2.png)

##### [3. CPU QoS说明](https://ashiamd.github.io/docsify-notes/#/study/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E5%92%8C%E7%A1%AC%E4%BB%B6/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E7%B3%BB%E7%BB%9F%E5%9B%9E%E9%A1%BE?id=_3-cpu-qos%e8%af%b4%e6%98%8e) {#_3-cpu-qos说明}

​ Hypervisor层根据**分时复用**的原理实现对VCPU的调度，**CPU QoS的原理是定期给各VCPU分配运行时间片，并对各VCPU运行的时间进行记账，对于消耗完时间片的虚拟CPU将被限制运行，直到获得时间片**。以此控制虚拟机获得物理计算资源的比例。以上分配时间片和记账的时间周期很短，对虚拟机用户来说会感觉一直在运行。

* CPU预留定义了分配给该VM的最少CPU资源。

* CPU限制定义了分配虚拟机占用CPU资源的上限。

* **CPU份额定义多个虚拟机在竞争CPU资源的时候按比例分配**。

* **CPU份额只在各虚拟机竞争计算资源时发挥作用，如果没有竞争，有需求的虚拟机可以独占主机的物理CPU资源**。

​如果虚拟机根据份额值计算出来的计算能力小于虚拟机预留值，调度算法会优先按照虚拟机预留值分配给虚拟机，对于预留值超出按份额分配的计算资源的部分，调度算法会从主机上其他虚拟机的CPU上按各自的份额比例扣除。

​**如果虚拟机根据份额值计算出来的计算能力大于虚拟机预留值，那么虚拟机的计算能力会以份额值计算为准**。以一台主频为2800MHz的单核物理机为例，如果满负载运行3台单VCPU的虚拟机A、B、C，分配情况如下。

| 主机CPU | VM | 份额 | 预留 | 按份额分配 | 最终分配 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 2800MHz | A | 1000 | 700MHz | 400MHz | 400+100+200=700MHz |
|  | B | 2000 | 0 | 800MHz | 800-100=700MHz |
|  | C | 4000 | 0 | 1600MHz | 1600-200=1400MHz |



