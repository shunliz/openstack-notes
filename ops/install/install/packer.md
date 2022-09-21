Packer 是一个优秀的开源镜像打包工具。Packer 的 builder 支持主流的公有云、私有云平台以及常见的虚拟化类型。同时它支持的 provisioner 能覆盖主流的配置管理工具: Ansible, Puppet, Chef, Windows Shell, Linux Shell 等。



在不可变的服务器的应用场景中，通过 Packer 自动创建虚拟机，然后调用 Ansible provisioner 从中央制品仓库拉取软件包、部署所需额外依赖包以及相关配置，最后自动打包成虚拟机镜像并回收该虚拟机资源，上传至云平台中。



