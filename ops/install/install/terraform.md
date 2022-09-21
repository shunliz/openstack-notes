# Terraform

它的一个重要功能之一，就是自动化地协助你云\(多云\)场景下的自动化基础设施资源的管理（AWS、谷歌云、阿里云等）。这意味着你可以以声明式的方式，自动化地创建、更新、删除你的公有云资源，减少手动点点点所带来地时间上地消耗和人工操作不可避免的风险.

1. 使用声明型语言HCL（HashiCorp Configuration Language）。使用户只关注与自己的需求，而非如何实现；
2. 采用客户端单一架构（Client Only），而非CS（Client/Server）架构。降低数据安全风险，同时提高了infrastructure配置的可靠性；
3. 完善的开源生态。只需要一个工具即可完成对多个云厂商的服务进行资源编排；

4. 设计（和测试）基础设施一次，然后多次重复使用该设计。

5. 配置文件可以是源代码控制的。

6. 在您喝 插入选择的饮料的同时，使用单个命令创建和破坏基础设施。

7. 监视状态更改（并即时实施设计更改）。

Terraform和Ansible在初识时应该如何定位他们？它们的定位都是自动化的工具，应该如何选择？

其实Terraform和Ansible是有功能重叠的。Terraform配置文件中的resource块，提供provisioner配置，连接到远程主机并进行类似ansible可执行的环境准备的操作。

但是可以明确的是，Terraform目前更合适你的基础设施创建和管理，如创建你的云主机、负载均衡器等等；Ansible而是更适合你的云主机创建后，自动化地去初始化你的机器配置、安装组件、部署服务等。

# Terraform如何工作？

Terraform通过社区或者其他人在Terraform Registry上公开的**provider**来调用云平台或各种服务的API接口，从而创建和管理资源。

![](/assets/other-deploy-terraform1.png)核心 Terraform 工作流程包含三个阶段：

Write-编写：定义资源，编写声明式配置文件定义资源。这些资源可能跨越多个云提供商和服务。

Plan-计划： Terraform 创建一个执行计划，描述它将根据现有基础架构和您的配置创建、更新或销毁的基础架构。

Apply-应用：在批准后，Terraform 会按照正确的顺序执行建议的操作，并尊重任何资源依赖关系。例如，如果您更新 VPC 的属性并更改该 VPC 中的虚拟机数量，Terraform 将在扩展虚拟机之前重新创建 VPC。

![](/assets/other-deploy-terraform2.png)        _Terraform默认情况下只管理通过Terraform创建出来的资源。但是实际生产中，我们一般都有了一些公有云的资源，才开始考虑使用Terraform来管理这些资源。_

_        实际这种场景下，需要使用terrform import将非terraform创建的资源进行导入。_

_        但是麻烦的是，每次只能导入一个资源。且terraform目前只接受这种方式来导入资源，并不能自动识别并生成相关配置。    
_

# ⭐关键概念

🍒Configuration：基础设施的定义和描述

基础设施即代码，其中的代码Code就是对基础设施资源的代码定义和描述，通过代码表达需要管理的资源。

所有资源的代码描述都是定义在一个以.tf结尾的文件，用于terraform的加载和解析。这个文件就称之为“Terraform模板”或者“configuration”

🍒Provider: 基础设施管理组件

Terraform常用于公有云上基础设施的管理，如虚拟机、网络、容器等。Provider就是与OpenAPI交互的后端驱动，Terraform通过Provider完成对基础设施资源的管理。

每个基础设施提供商，aliyun、aws等都需要提供一个provider来实现对自家资源的统一管理。目前我们使用的阿里云对应的provider就是alicloud。

在运行环境中，Terraform和Provider是两个独立存在的package，执行Terraform时，会根据用户模板中指定的Provider或者resource/datasource的标志自动下载模板使用的provider，并放在当前目录下的.terraform隐藏目录下。

🍒Resource：基础设施资源和服务的管理

在Terraform中，一个具体的资源或者服务称为resource，比如一个ECS，一个SLB、一个域名解析记录。每个特定的resource包含了若干可用于描述对应资源或服务的属性字段。通过这些字段来定义一个完整的资源或者服务，比如dns的domain\_name、ttl等。

如下定义一个resource：

```
resource "alicloud_alidns_record" "dns701438486351555584" {
domain_name = "test.com"
line = "default"
priority = 0
rr = "mobile.api"
status = "ENABLE"
ttl = 600
type = "A"
value = "1.1.1.4"
}
```

其中alicloud\_alidns\_record为资源类型，定义这个资源的类型，告诉terraform这个resource是域名解析记录。

dns701438486351555584为资源名称，资源名称在同一个模板中必须唯一，可以用于其他资源引用该资源。

大括号里面的block为配置参数，定义资源的属性。

🍒Data Source：基础设施资源和服务的查询

Data Source提供查询资源的功能，每个data source实现对一个资源的动态查询，其结果可以认为是动态变量，只有运行时才知道其值。

```
data "alicloud_alidns_records" "records_ds_uni" {
domain_name = "test.com"
type = "A"
line = "unicom"
rr_regex = "mobile*.api"
output_file = "records-uni.txt"
}
```

如上定义一个records\_ds\_uni的资源，其通过data引用，查询test.com域名下，解析记录匹配mobile\*.api的，解析线路为unicom的所有A记录，并输出到records-uni.txt文本中。

🍒state：保存资源关系以及属性文件的数据库

Terraform创建和管理所有资源都保存在自己的数据库上，这个数据库是一个名为terraform.tfstate文件，在terraform中称之为state，默认存放在执行命令的本地目录中。

在执行terraform命令时，terraform会利用state文件与模板文件进行diff对比，如果出现不一致，terraform将按照模板中的定义重新创建，或者修改资源，直到没有diff。所以这个文件非常重要，如果损坏，terraform将认为已创建的资源被破坏，或者需要重建。当然实际的云资源不会收到影响。

🍒Backend：存储state文件的载体

因terraform创建资源后，会将资源属性保存在state文件中，而这个文件可以放本地，也可以存放在远端，实现state和模板代码的分离，这个存放state文件的载体就是backend。

Backend分为本地和remote两类，默认为本地。目前已支持多达13中远端存储方案，如console、etcd、oss等，可以降低多人协作对state维护的成本，也可以保障数据的安全性。

🍒Provisioner：在机器上执行操作的组件

用来在本地机器或者登录远程主机执行相关的操作，如local-exec在本地执行命令，chef用来在远程主机安装、配置、执行chef client，remote-exec用来登录远程主机执行命令。

通常与provider搭配实现，provider创建资源后，使用provisioner在创建的资源上执行各种操作。

# 🍒常用命令

terraform init: 初始化，加载所需模块

terraform plan： 资源预览

用于对模板定义的资源进行预览。如预览当前模板中定义的资源是否符合预期，如果存在state文件则展示diff结果，即变更的内容。

terraform apply：新建、变更资源

terraform show：资源展示，展示当前state中所管理的资源以及所有属性

terraform destroy： 资源释放

terraform import： 资源导入，将存量的云资源导入到state中，进而加入到terraform的管理体系中。适用以下场景：

从来没使用terraform管理过资源，现在需要切换到terraform管理；

在不影响资源使用的前提下，重构资源模板中的定义；

Provider有升级支持了更多的参数，需要把新参数同步过来。

terraform fmt： 格式化模板文件。将编写的tf文件进行就地格式化。

![](/assets/other-deploy-terraform3.png)

# 操作的生命周期

![](/assets/other-deploy-terraform4.png)

资源编排的动作的生命周期如上，其中左侧为Terraform系统系统的能力，右侧provider、provisioner为厂商提供。

当执行terraform apply命令时：

①、terraform唤醒进程，初始化backend\(默认为local-file\)；

②、解析用户定义的模板文件，并获取最新的资源状态，进行对比；

③、构建DAG，将所有编排动作依次发送给provider；

④、provider调用云API管理云资源

⑤、将返回的结果写回state





