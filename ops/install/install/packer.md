Packer 是一个优秀的开源镜像打包工具。Packer 的 builder 支持主流的公有云、私有云平台以及常见的虚拟化类型。同时它支持的 provisioner 能覆盖主流的配置管理工具: Ansible, Puppet, Chef, Windows Shell, Linux Shell 等。

在不可变的服务器的应用场景中，通过 Packer 自动创建虚拟机，然后调用 Ansible provisioner 从中央制品仓库拉取软件包、部署所需额外依赖包以及相关配置，最后自动打包成虚拟机镜像并回收该虚拟机资源，上传至云平台中。

## 1 简介 {#1-简介}

* 基于相同的配置，为多个云平台生成不同的服务器镜像
* 可以创建匹配不同云平台的服务器镜像，例如aws的ami，VMware的VMDK / VMX文
* 性能强劲，可以为多个云平台并发创建服务器镜像
* 可以很方便集成chef/puppet等配置服务

## 2 为什么使用packer {#2-为什么使用packer}

* 各平台的镜像创建较为繁琐
* packer使用较为简单，可以创建多种云平台的镜像

优点：

* 超级快的部署和配置虚拟机：几秒的时间完成计算机申请、部署和配置工作
* 跨云平台服务商支持：支持aws/openstack/vmware等云平台服务商，非常快速满足开发/测试/生产的需求
* 稳定性提升：创建和配置所有的软件，如果有问题，可以提前发现
* 可测试性更高：能快速启动服务器用于各种测试，提升可测试性

## 3 使用场景 {#3-使用场景}

* 服务器镜像持续交付工具
* 基于相同模板可以创建满足开发/测试/生产等不同运行环境的镜像，例如开发环境跑openstack、生产环境跑aws，基于相同的packer模板可以一键创建满足不同环境需求的服务器镜像
* 适合快速创建演示产品，基于生产的模板，快速在一个地方进行搭建，并进行功能演示

## 4 使用 {#4-使用}

### 创建aws ec2 ami {#创建aws-ec2-ami}

* 创建模板：模板为json格式
* 模板包含：variables\(外部变量\)、builder\(构建方法\)等
* build：通过packer build template构建模板，生成ami
* 管理镜像：通过以上，会在aws账号生成对应的ami镜像，可以后续自行管理，例如删除等
* provisioner：用于镜像构建后执行一些操作，操作类型可以是shell，也可以是将文件创建到镜像中
* 构建：构建完成后，将会生成一个ami，并挂在对应的账户下

## 5 Provision {#5-provision}

* provision实现了基于原始的镜像进行重新配置的能力
* 例如支持在原有镜像中引入一个新的服务

## 6 并发构建 {#6-并发构建}

* 支持在一个packer模板中并发构建面向不同云服务商的镜像
* 例如同时构建aws的ami镜像和vmware的虚拟机镜像
* variables：可以把多个平台用到的变量一并生命
* build：多种build时，支持多个build构建不同平台的镜像
* provision：多个平台的镜像build完成后，可以同时执行同一个provision
* 构建：构建完成后，会创建两个虚拟机镜像， 一个挂载aws的账户下，另一个挂在vmware的平台上

## 7 实例

    {
         "variables": {
           "access_key": "{{env `ALICLOUD_ACCESS_KEY`}}",
           "secret_key": "{{env `ALICLOUD_SECRET_KEY`}}"
         },
         "builders": [{
           "type":"alicloud-ecs",
           "access_key":"{{user `access_key`}}",
           "secret_key":"{{user `secret_key`}}",
           "region":"cn-beijing",
           "image_name":"packer_basic",
           "source_image":"centos_7_02_64_20G_alibase_20170818.vhd",
           "ssh_username":"root",
           "instance_type":"ecs.n1.tiny",
           "internet_charge_type":"PayByTraffic",
           "io_optimized":"true"
         }],
         "provisioners": [{
           "type": "shell",
           "inline": [
             "sleep 30",
             "yum install redis.x86_64 -y"
           ]
         }]
       }

```
指定Packer模板文件生成自定义镜像的操作步骤如下：

运行命令export ALICLOUD_ACCESS_KEY=<您的AccessKeyID>导入您的AccessKeyID。
运行命令export ALICLOUD_SECRET_KEY=<您的AccessKeySecret>导入您的AccessKeySecret。
运行命令packer build alicloud.json创建自定义镜像。
```

| 参数 | 描述 |
| :--- | :--- |
| access\_key | 您的AccessKeyID。更多详情，请参见[创建AccessKey](https://help.aliyun.com/document_detail/53045.html#concept-53045-zh)。 **说明** 由于AccessKey权限过大，为防止错误操作，建议您创建RAM用户，并使用RAM子账号创建AccessKey。具体步骤，请参见[创建 RAM 用户](https://help.aliyun.com/document_detail/28637.html#concept-gpm-ccf-xdb)和[创建AccessKey](https://help.aliyun.com/document_detail/53045.html#concept-53045-zh)。 |
| secret\_key | 您的AccessKeySecret。更多详情，请参见[创建AccessKey](https://help.aliyun.com/document_detail/53045.html#concept-53045-zh)。 |
| region | 创建自定义镜像时使用临时资源的地域。 |
| image\_name | 自定义镜像的名称。 |
| source\_image | 基础镜像的名称，可以从阿里云公共镜像列表获得。 |
| instance\_type | 创建自定义镜像时生成的临时实例的类型。 |
| internet\_charge\_type | 创建自定义镜像时临时实例的公网带宽付费类型。 |
| provisioners | 创建自定义镜像时使用的Packer配置器类型。详情请参见[Packer配置器](https://www.packer.io/docs/provisioners/index.html)。 |



