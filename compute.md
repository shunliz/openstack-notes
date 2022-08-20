从云计算整个大框架分类方式来看，计算部分我们也大致可以分为硬件和软件两部分，但是硬件部分其实就是传统计算机系统，包括CPU，内存，IO等系统，这部分已经有很多经典教材了，比如《深入理解计算机系统》，当然还包括大名鼎鼎的Intel CPU手册了。但是这部分不是我们的重点，后边我们主要从软件方面介绍云计算相关技术，比如 KVM， 容器技术等。



EC2 Amazon Nitro System![](/assets/compute-virtualization-maturity-level.png)Amazon Nitro System 不仅仅是一个单一的专用硬件设备，而是一套完整的软硬件融合协同系统。由 Nitro Hypervisor、Nitro I/O Accelerator Cards、Nitro Security Chip 三大部分组成，今年又再继续加入了 Nitro Enclaves 和 NitroTPM（Coming soon in 2022）两大组成部分。彼此合作，又相互独立。

![](/assets/compute-ec2-nitro-system.png)

**Nitro Hypervisor**：是一个 Lightweight Hypervisor（轻量级虚拟机监控程序）只负责管理 CPU 和 Memory 的分配，几乎不占用 Host 资源，所有的服务器资源都可用来执行客户的工作负载。

* **Nitro Cards**：是一系列用于 Offloading and Accelerated（卸载和加速）的协处理外设卡，承载网络、存储、安全及管理功能，使得网络和存储性能得到了极大提升，并且从硬件层提供天然的安全保障。

* **Nitro Security Chip**：随着更多的功能被卸载到专用硬件设备上，Nitro Security Chip 提供了面向专用硬件设备及其固件的安全防护能力，包括限制云平台维护人员对设备的访问权限，消除人为的错误操作和恶意篡改。

* **Nitro Enclaves**：为了进一步保障 EC2 用户的个人信息保护及数据安全，Nitro Enclaves 基于 Nitro Hypervisor 进一步提供了创建 CPU 和 Memory 完全隔离的计算环境的能力，以保护和安全地处理高度敏感的数据。

* **Nitro TPM（Trusted Platform Module，可信平台模块）**：支持 TPM 2.0 标准，Nitro TPM 允许 EC2 实例生成、存储和使用密钥，继而支持通过 TPM 2.0 认证机制提供实例完整性的加密验证，能够有效地保护 EC2 实例，防止非法用户访问用户的个人隐私数据。



