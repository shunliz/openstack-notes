## 项目简介 {#项目简介}

cloud-init是一款用于初始化云服务器的工具，它拥有丰富的模块，能够为云服务器提供的能力有：初始化密码、扩容根分区、设置主机名、注入公钥、执行自定义脚本等等，功能十分强大。

目前为止cloud-init是云服务器初始化工具中的事实标准，它几乎适用于所有主流的Linux发行版，也是各大云厂商正在使用的默认工具，社区活跃。基于Python语言使得它能够轻易跨平台、跨架构运行，良好的语法抽象使得它适配新模块、新发行版十分容易。

## 基本概念 {#基本概念}

### 实例数据与数据源 {#实例数据与数据源}

实例数据（Instance data）是cloud-init用来处理并配置实例的数据集合。根据用途，可以将实例数据划分为如下三类：

* metadata：一系列字典格式的元数据，用作模板渲染、模块运行等等。
* userdata：启动实例时用户能够指定的数据（单实例数据）
* vendordata：云基座传入的数据（全局数据）

在openstack中，metadata、userdata均在用户创建实例时通过相应参数传入，而vendordata则由nova-api-metadata服务启动时指定。

![](/assets/compute-nova-cloud-init-metadata1.png)

这些数据可能来源于许多地方，cloud-init使用**数据源**一词表示实例数据的来源，目前内置的数据源有：

```
"NoCloud",
"ConfigDrive",
"LXD",
"OpenNebula",
"DigitalOcean",
"Azure",
"AltCloud",
"OVF",
"MAAS",
"GCE",
"OpenStack",
"AliYun",
"Vultr",
"Ec2",
"CloudSigma",
"CloudStack",
"SmartOS",
"Bigstep",
"Scaleway",
"Hetzner",
"IBMCloud",
"Oracle",
"Exoscale",
"RbxCloud",
"UpCloud",
"VMware",
"None",
```

等等，不同的数据源也表明了不同的实例数据搜索方式。cloud-init在实例内部启动时并不知道从哪里才能够找到实例数据，它会根据预设的一个数据源的列表一个一个查找实例数据，而首个能够找到实例数据的数据源将成为这次启动的数据源。

![](/assets/compute-nova-cloud-init-datasroucesearch.png)

如果cloud-init指定了OpenStack数据源，那么实例数据将均通过HTTP API请求获取，不同类别的实例数据有着不同的获取方式：

* metadata：
  [http://169.254.169.254/openstack/latest/meta\_data.json](http://169.254.169.254/openstack/latest/meta_data.json)
* userdata：
  [http://169.254.169.254/openstack/latest/user\_data](http://169.254.169.254/openstack/latest/user_data)
* vendordata：
  [http://169.254.169.254/openstack/latest/vendor\_data.json、http://169.254.169.254/openstack/latest/network\_data.json](http://169.254.169.254/openstack/latest/vendor_data.json%E3%80%81http://169.254.169.254/openstack/latest/network_data.json)

而如果指定了ConfigDrive数据源，那么实例数据将均通过文件获取，cloud-init会搜索标记有**CONFIG-2**标签的分区，随后将该分区挂载至临时目录中，并读取其中包含实例数据的文件：

* metadata：file://TMPDIR/openstack/latest/meta\_data.json
* userdata：file://TMPDIR/openstack/latest/user\_data
* vendordata：file://TMPDIR/openstack/latest/vendor\_data.json、file://TMPDIR/openstack/latest/network\_data.json

所有数据源的参考文档：[Datasources — cloud-init 22.1 documentation \(cloudinit.readthedocs.io\)](https://cloudinit.readthedocs.io/en/latest/topics/datasources.html)

### 启动阶段 {#启动阶段}

在整个系统启动的过程中，cloud-init的执行包括5个阶段，执行阶段从前到后分别为：Generator、Local、Network、Config、Final，每一个阶段都有着它们各自的作用。![](/assets/compute-nova-cloud-init-generator.png)Generator

* systemd服务：/usr/lib/systemd/system-generators/cloud-init-generator
* 运行于：系统刚启动时

generator是最先运行的阶段，它的功能包括：

1. 判断是否需要禁止运行cloud-init。generator会根据如下条件判断：
   1. /etc/cloud/cloud-init.disabled文件是否存在。
   2. 内核参数中是否包括有cloud-init=disabled配置项。
2. 筛选可用的数据源。generator为每个数据源都写了判断函数，运行判断函数后返回可用数据源的列表，存放于datasource\_list配置项并写入/run/cloud-init/cloud.cfg中。

部分情况下，generator对/run/cloud-init/cloud.cfg文件的写入会对cloud-init的运行产生影响，此时可以通过修改/etc/cloud/ds-identify.cfg配置文件，从而更改generator的行为。

#### Generator {#generator}

* systemd服务：/usr/lib/systemd/system-generators/cloud-init-generator
* 运行于：系统刚启动时

generator是最先运行的阶段，它的功能包括：

1. 判断是否需要禁止运行cloud-init。generator会根据如下条件判断：
   1. /etc/cloud/cloud-init.disabled文件是否存在。
   2. 内核参数中是否包括有cloud-init=disabled配置项。
2. 筛选可用的数据源。generator为每个数据源都写了判断函数，运行判断函数后返回可用数据源的列表，存放于datasource\_list配置项并写入/run/cloud-init/cloud.cfg中。

部分情况下，generator对/run/cloud-init/cloud.cfg文件的写入会对cloud-init的运行产生影响，此时可以通过修改/etc/cloud/ds-identify.cfg配置文件，从而更改generator的行为。

```
# /etc/cloud/ds-identify.cfg
# 该策略表示启动cloud-init，但不筛选可用数据源
policy: enabled,found=all,maybe=none,notfound=disabled
```

#### Local {#local}

* systemd服务：/usr/lib/systemd/system/cloud-init-local.service
* 运行于：根目录挂载并可读写后

Local阶段的主要功能为：

1. 搜索数据源。在datasource\_list配置项中搜索第一个可用的数据源，作为本次启动的数据源。数据源搜索细节如下：
   1. 尝试从缓存恢复。若存在obj.pkl缓存，为trust模式或ds.check\_instance\_id\(\)返回true，则从缓存恢复。
   2. 搜索数据源，遍历datasource\_list配置项中的数据源，找出第一个可用的数据源，并返回。
2. 应用网络配置（不拉起网络）。网络配置的来源如下：
   1. 数据源，使用从数据源获取的网络配置，渲染并写入网络配置至磁盘中。
   2. 回退，如果无数据源，则写入一个默认的dhcp网络配置。
   3. 不应用，如果cloud-init配置文件中有如下配置项：network: {config: disabled}，或者存在/var/lib/cloud/data/upgraded-network文件，则不应用网络配置。

#### Network {#network}

* systemd服务：/usr/lib/systemd/system/cloud-init.service
* 运行于：网络服务启动之后

Network阶段的主要功能为：

1. self.datasource.setup\(\)，在uesr-data和vendor-data处理之前调用，用于网络启动后再次更新数据源，目前仅用于azure获取fabric数据并填充进fabric\_data。
2. 存储与渲染userdata和vendor\_data。
3. self.datasource.activate\(\)，该方法在user-data和vendor-data渲染后，init\_modules执行前调用。
4. 运行cloud\_init\_modules中配置的模块。

#### Config {#config}

* systemd服务：/usr/lib/systemd/system/cloud-config.service
* 运行于：Network阶段之后

Config阶段的主要功能是运行cloud\_config\_modules中配置的模块。

#### Final {#final}

* systemd服务：/usr/lib/systemd/system/cloud-final.service
* 运行于：Config阶段之后

Config阶段的主要功能是运行cloud\_final\_modules中配置的模块。

### cloud-config {#cloud-config}

cloud-config是cloud-init的配置文件，它用于控制cloud-init的行为，比如说该运行哪些功能，以及每个功能如何运行等等。和其他软件的配置文件相比，cloud-config具有高度定制化的特点，除了云服务器本身的cloud-config以外，cloud-init还能够从vendordata、userdata甚至内核参数中获取cloud-config，从而使得用户能够方便地利用cloud-init定制自己的云服务器。![](/assets/compute-nova-cloud-init-config.png)模块

模块（Modules）是cloud-init运行的主体，所有我们需要的用来初始化云服务器的功能都是通过执行模块而实现的。以**set\_hostname**模块为例，该模块的功能是设置主机名，当运行该模块时，它会读取cloud-config中的hostname参数，并将其中值设置为云实例的主机名。

![](/assets/compuute-nova-cloud-init-modules.png)

每个模块都有名称、运行频率、配置参数这三大要素，其中运行频率表示一个模块该在什么时候运行，通常有两种运行频率：1. once-per-instance，表示仅在实例首次启动时运行；2. always，表示实例每次启动都运行。

是否运行某个模块、何时运行模块、怎么运行模块，这些都可以在cloud-config中配置，下面截取部分cloud-config，可以看到在这个云实例中：

* network阶段将运行ssh模块，config阶段将运行mounts、locale等模块，final阶段将运行scripts开头的一些模块；
* set-passwords模块默认的运行频率是once-per-instance，可将其调整为always，表示每次启动云实例时都会设置密码；
* ssh\_pwauth是set-passwords模块的配置项，ssh\_deletekeys、ssh\_genkeytypes是ssh模块的配置项，它们描述了该如何运行模块。

```
# /etc/cloud/cloud.cfg
ssh_pwauth: 1
ssh_deletekeys: 1
ssh_genkeytypes: ['rsa', 'ecdsa', 'ed25519']

cloud_init_modules:
 - ssh

cloud_config_modules:
 - mounts
 - locale
 - [set-passwords,always]

cloud_final_modules:
 - scripts-per-once
 - scripts-per-boot
 - scripts-per-instance
 - scripts-user
```

所有模块的参考文档：[https://cloudinit.readthedocs.io/en/latest/topics/modules.html](https://cloudinit.readthedocs.io/en/latest/topics/modules.html)

### 发行版 {#发行版}

cloud-init设计支持绝大多数的主流linux发行版，诸如Centos、Ubuntu、Arch、Fedora等等，然而不同的linux发行版可能在使用方式上有着或多或少的不同，因此cloud-init需要针对不同的发行版进行一定程度的抽象，形成一个统一层并提供给上层的模块使用。就算是这样，某些模块也只能提供给特定的发行版使用，比如apk\_configure模块只能alpine使用，而apt\_configure模块则只能在ubuntu、debian上使用，这点在使用cloud-init的过程中需要稍加注意。

## 基础用法 {#基础用法}

我们在使用cloud-init时一般不会特别地去执行cloud-init相关的命令，在云实例的启动过程中操作系统会按正常流程去执行cloud-init，不过当需要开发/调试cloud-init的时候，这些命令会带来很大的帮助。

执行cloud-init的四个阶段：

```
# local阶段
cloud-init init --local
# network阶段
cloud-init init
# config阶段
cloud-init modules --mode=config
# final阶段
cloud-init modules --mode=final

```

查询相关：

```
# 查询cloud-id
cloud-id
# 查询cloud-init执行状态
cloud-init status -l
# 查询metadata
cloud-init query <variable>
cloud-init query -l
```

清理缓存：

```
cloud-init clean
rm -rf /var/run/cloud-init/
rm -rf /var/lib/cloud/
rm -rf /var/log/cloud-init.log
rm -rf /etc/sysconfig/network-scripts/ifcfg-*
```

其他命令：

```
# 创建一个包含cloud-config(config.yaml)和shell脚本(script.sh)的userdata
cloud-init devel make-mime -a config.yaml:cloud-config -a script.sh:x-shellscript > userdata
```

使用案例

### 重置密码 {#重置密码}

cloud-config方式：

```
##template: jinja
#cloud-config

runcmd:
 # OpenStack数据源在local阶段无法获取数据，network阶段由于是trust模式，直接从缓存读取数据，无法获取更新后metadata，因此需要事先删除缓存。
 - DIR=/var/lib/cloud/scripts/per-boot/ && mkdir -p $DIR && cd $DIR && echo '#!/bin/bash' > 99-clean-cloudinit-cache.sh && echo 'rm -f /var/lib/cloud/instance/obj.pkl' >> 99-clean-cloudinit-cache.sh && chmod +x 99-clean-cloudinit-cache.sh
 # ConfigDrive数据源无法获取更新后metadata，需在首次启动后修改为OpenStack数据源。
 - echo 'datasource_list: [OpenStack]' > /etc/cloud/cloud.cfg.d/90_openstack_datasource.cfg
 # 首次启动后禁止写入网络配置，防止ConfigDrive中的bond配置被覆盖。
 - touch /var/lib/cloud/data/upgraded-network

{% if ds.meta_data.meta.admin_pass is defined %}
chpasswd:
  list: |
    root:{{ds.meta_data.meta.admin_pass}}
  expire: false
  ssh_pwauth: [true]

phone_home:
  url: http://169.254.169.254/openstack/latest/phone_home?unset=admin_pass
  post:
    - instance_id
  tries: 5

cloud_config_modules:
 - [set-passwords,always]
 - [phone_home,always]
 - ssh
 - runcmd
{% endif %}
{% endif %}
```

获取更新后metadata的方法，除删除缓存之外，也可实现setup方法，在该方法中获取metadata并填充进数据源中（参考azure数据源）。

将set-passwords和phone\_home模块设为always的方法，出了在vendor-data中指定外，也可在activate方法中实现，删除执行这两个模块的信号（参考exoscale数据源）。

### ssh公钥注入 {#ssh公钥注入}

使用如下userdata注入ssh公钥：

```
##template: jinja
#cloud-config

ssh_authorized_keys:
  - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDIe6oWYfMB4m9TDSmyVlONj2UakYKkAnpTsZnYXWHUd6zLQDvCLkeXmLiBTuJ51Dh6nolL4rZETj1wLJpXqcMvDhhOZyA0Ji3ENbeJj0URWNjDpRUc3eUApDOzpa3gHm8yRwXDZsZbFMpqcL5vQjlZJ9K+SmLCvbfk6sG04uJWLsIFrjSkxlRxdWwHwdvMH9tAo3UeQGnzRxdnF4zxrob+vAV7PPqJ9u4J6GwXWykTtS+cmvoPOBWQHL5gULxp2FVt/xTF5xY2IRHpzfNmJlCd4G7HGow/PVD+jsIN57iNwvxU7dvTzg0PVUXGQImJvQ8pkHD6//ghXcLleUQz3f+7Brlc6yEZuDdEy2NFEnYW8CoEymPSXVZaBykiggP2XSGp6gQa1gtT7yy44EEB5kjX9epi+jAwiCRaLDrgSLBWQmJp8435dtoOALZOXb1Br0uOA0a3G3KZr+7v11f09NZgFPzJTrY2d4ZNtOUILOcQkUAhtW6yOwuCiuxVeKM8cjE= webssh@hikcloud
```



## 源码分析 {#源码分析}

### 代码结构 {#代码结构}

cloud-init的代码结构如下：

```
cloud-init
├── bash_completion # bash自动补全文件
│   └── cloud-init
├── ChangeLog # 更新日志
├── cloudinit
│   ├── cloud.py # Cloud类
│   ├── cmd # 命令行操作目录
│   │   ├── clean.py # cloud-init clean
│   │   ├── cloud_id.py # cloud-id
│   │   ├── devel # cloud-init devel
│   │   ├── __init__.py
│   │   ├── main.py # cloud-init init/modules
│   │   ├── query.py # cloud-init query
│   │   └── status.py # cloud-init status
│   ├── config # 模块目录
│   ├── distros # 系统发行版目录
│   ├── handlers # 模板渲染处理函数目录
│   ├── helpers.py # 帮助函数
│   ├── __init__.py
│   ├── log.py # 日志处理
│   ├── mergers # 配置合并目录
│   ├── net # 网络目录
│   ├── settings.py # 内置配置
│   ├── sources #数据源目录
│   ├── stages.py # Init类
│   ├── ...
│   └── warnings.py
├── doc # 文档
│   ├── ...
├── packages # 各大Linux发行包制作脚本
│   ├── ...
├── README.md # 简介
├── requirements.txt # 依赖包
├── rhel # 针对redhat系linux发行版的补丁
│   ├── cloud.cfg
│   ├── cloud-init-tmpfiles.conf
│   ├── README.rhel
│   └── systemd
├── setup.py # python模块安装文件
├── tests # 单元测试目录
│   ├── ...
├── tools # 额外工具
│   ├── ...
└── tox.ini # tox配置文件

44 directories, 127 files
```

cloud-init的函数入口点位于cloudinit/cmd/main.py，该文件包含了所有cloud-init执行阶段的逻辑代码。

### cloud-init init \[ --local \] {#cloud-init-init----local-}

cmd/main.py，main\_init：init阶段运行此主函数，分析如下：

```py
def main_init(name, args):
    # deps变量，local阶段为NETWORK，network阶段为FILESYSTEM、NETWORK
    deps = [sources.DEP_FILESYSTEM, sources.DEP_NETWORK]
    if args.local:
        deps = [sources.DEP_FILESYSTEM]
	# 标准输出log
    early_logs = [attempt_cmdline_url(
        path=os.path.join("%s.d" % CLOUD_CONFIG,
                          "91_kernel_cmdline_url.cfg"),
        network=not args.local)]
	# 决定欢迎语，local阶段为init-local，network阶段为init
    if not args.local:
        w_msg = welcome_format(name)
    else:
        # Cloud-init v. 19.4 running 'init-local' at Wed, 15 Dec 2021 03:00:40 +0000. Up 250.20 seconds.
        w_msg = welcome_format("%s-local" % (name))
    # 实例化stages.Init类
    init = stages.Init(ds_deps=deps, reporter=args.reporter)
    # Stage 1
    # 加载配置文件，优先级从低到高为：内置配置 --> /etc/cloud/cloud.cfg{,.d} --> /run/cloud-init/cloud.cfg --> kernel cmdline
    init.read_cfg(extract_fns(args))
    # Stage 2
    # 重定向输出和错误
    outfmt = None
    errfmt = None
    try:
        early_logs.append((logging.DEBUG, "Closing stdin."))
        util.close_stdin()
        (outfmt, errfmt) = util.fixup_output(init.cfg, name)
    except Exception:
        msg = "Failed to setup output redirection!"
        util.logexc(LOG, msg)
        print_exc(msg)
        early_logs.append((logging.WARN, msg))
    if args.debug:
        # Reset so that all the debug handlers are closed out
        LOG.debug(("Logging being reset, this logger may no"
                   " longer be active shortly"))
        logging.resetLogging()
    logging.setupLogging(init.cfg)
    apply_reporting_cfg(init.cfg)

    # Any log usage prior to setupLogging above did not have local user log
    # config applied.  We send the welcome message now, as stderr/out have
    # been redirected and log now configured.
    # 输出欢迎语
    welcome(name, msg=w_msg)

    # re-play early log messages before logging was setup
    for lvl, msg in early_logs:
        LOG.log(lvl, msg)

    # Stage 3
    try:
        # 创建cloud-init相关的目录和文件，包括/var/lib/cloud/目录下的各个子目录，以及日志文件
        init.initialize()
    except Exception:
        util.logexc(LOG, "Failed to initialize, likely bad things to come!")
    # Stage 4
    # 判断manual_cache_clean配置项，如果为false，cloudinit会通过实例id判断当前运行的实例是否为新实例；否则不作判断，可能导致实例迁移后per-instance模块不运行
    # local阶段，删除缓存（boot_finished、no-net）
    # network阶段，判断no-net文件是否存在，如存在则提前退出
    path_helper = init.paths
    mode = sources.DSMODE_LOCAL if args.local else sources.DSMODE_NETWORK

    if mode == sources.DSMODE_NETWORK:
        existing = "trust"
        sys.stderr.write("%s\n" % (netinfo.debug_info()))
        LOG.debug(("Checking to see if files that we need already"
                   " exist from a previous run that would allow us"
                   " to stop early."))
        # no-net is written by upstart cloud-init-nonet when network failed
        # to come up
        stop_files = [
            os.path.join(path_helper.get_cpath("data"), "no-net"),
        ]
        existing_files = []
        for fn in stop_files:
            if os.path.isfile(fn):
                existing_files.append(fn)

        if existing_files:
            LOG.debug("[%s] Exiting. stop file %s existed",
                      mode, existing_files)
            return (None, [])
        else:
            LOG.debug("Execution continuing, no previous run detected that"
                      " would allow us to stop early.")
    else:
        existing = "check"
        mcfg = util.get_cfg_option_bool(init.cfg, 'manual_cache_clean', False)
        if mcfg:
            LOG.debug("manual cache clean set from config")
            existing = "trust"
        else:
            mfile = path_helper.get_ipath_cur("manual_clean_marker")
            if os.path.exists(mfile):
                LOG.debug("manual cache clean found from marker: %s", mfile)
                existing = "trust"

        init.purge_cache()
        # Delete the no-net file as well
        util.del_file(os.path.join(path_helper.get_cpath("data"), "no-net"))

    # Stage 5
    # 从数据源中获取数据。根据obj.pkl缓存文件是否存在、existing变量、instance_id是否与/run/cloud-init/instance-id一致判断是否从缓存加载数据，
    # 否则遍历所有数据源，选择能够第一个能够获取数据的数据源当作本实例数据源
    # s.update_metadata([EventType.BOOT_NEW_INSTANCE])，_get_data
    try:
        init.fetch(existing=existing)
        # if in network mode, and the datasource is local
        # then work was done at that stage.
        # network阶段下，如果数据源的dsmode不为network，则直接结束
        if mode == sources.DSMODE_NETWORK and init.datasource.dsmode != mode:
            LOG.debug("[%s] Exiting. datasource %s in local mode",
                      mode, init.datasource)
            return (None, [])
    except sources.DataSourceNotFoundException:
        # In the case of 'cloud-init init' without '--local' it is a bit
        # more likely that the user would consider it failure if nothing was
        # found. When using upstart it will also mentions job failure
        # in console log if exit code is != 0.
        if mode == sources.DSMODE_LOCAL:
            LOG.debug("No local datasource found")
        else:
            util.logexc(LOG, ("No instance datasource found!"
                              " Likely bad things to come!"))
        if not args.force:
            init.apply_network_config(bring_up=not args.local)
            LOG.debug("[%s] Exiting without datasource", mode)
            if mode == sources.DSMODE_LOCAL:
                return (None, [])
            else:
                return (None, ["No instance datasource found."])
        else:
            LOG.debug("[%s] barreling on in force mode without datasource",
                      mode)

    # 如果数据源是从缓存恢复的，且instance-data.json文件缺失，则恢复它
    _maybe_persist_instance_data(init)
    # Stage 6
    # 生成/var/lib/cloud/<instance_id>/软链接，创建handlers, scripts, sem目录，写入datasource文件
    # 在/var/lib/cloud/data/目录下写入previous-datasource、instance-id、previous-instance-id文件
    # 在/run/cloud-init/目录下写入.instance_id文件
    # 若manual_cache_clean配置项为true，写入/var/lib/cloud/<instance_id>/manual_clean_marker文件
    # 写入obj.pkl
    # 刷新init实例的配置
    iid = init.instancify()
    LOG.debug("[%s] %s will now be targeting instance id: %s. new=%s",
              mode, name, iid, init.is_new_instance())

    if mode == sources.DSMODE_LOCAL:
        # Before network comes up, set any configured hostname to allow
        # dhcp clients to advertize this hostname to any DDNS services
        # LP: #1746455.
        _maybe_set_hostname(init, stage='local', retry_stage='network')
    # 应用网络配置，network阶段会拉起网络
    # 若存在/var/lib/cloud/data/upgraded-network文件，则直接返回
    # netcfg=self.datasource.network_config
    # self._apply_netcfg_names(netcfg)
    # self.distro.apply_network_config(netcfg, bring_up=bring_up)
    init.apply_network_config(bring_up=bool(mode != sources.DSMODE_LOCAL))

    # local阶段下，如果数据源的dsmode不为local，则直接返回
    if mode == sources.DSMODE_LOCAL:
        if init.datasource.dsmode != mode:
            LOG.debug("[%s] Exiting. datasource %s not in local mode.",
                      mode, init.datasource)
            return (init.datasource, [])
        else:
            LOG.debug("[%s] %s is in local mode, will apply init modules now.",
                      mode, init.datasource)

    # Give the datasource a chance to use network resources.
    # This is used on Azure to communicate with the fabric over network.
    # 在uesr-data和vendor-data处理之前调用，用于网络启动后再次更新数据源，目前仅用于azure获取fabric数据并填充进fabric_data
    # 调用self.datasource.setup(is_new_instance=self.is_new_instance())
    init.setup_datasource()
    # update fully realizes user-data (pulling in #include if necessary)
    # 存储与渲染userdata和vendor_data
    # _store_userdata()，在/var/lib/cloud/instance/目录下写入user-data.txt、user-data.txt.i
    # _store_vendordata()，在/var/lib/cloud/instance/目录下写入vendor-data.txt、vendor-data.txt.i
    init.update()
    _maybe_set_hostname(init, stage='init-net', retry_stage='modules:config')
    # Stage 7
    try:
        # Attempt to consume the data per instance.
        # This may run user-data handlers and/or perform
        # url downloads and such as needed.
        # 消费uesr_data和vendor_data
        # allow_userdata不为false的话，执行_consume_userdata(PER_INSTANCE)，reading and applying user-data
        # 在/var/lib/cloud/instance/目录下写入cloud-config.txt
        # 执行_consume_vendordata(PER_INSTANCE)，vendor data will be consumed
        # 在/var/lib/cloud/instance/目录下写入vendor-cloud-config.txt
        # 在/var/lib/cloud/instance/scripts/vendor/目录下写入vendor_data脚本
        (ran, _results) = init.cloudify().run('consume_data',
                                              init.consume_data,
                                              args=[PER_INSTANCE],
                                              freq=PER_INSTANCE)
        if not ran:
            # Just consume anything that is set to run per-always
            # if nothing ran in the per-instance code
            #
            # See: https://bugs.launchpad.net/bugs/819507 for a little
            # reason behind this...
            init.consume_data(PER_ALWAYS)
    except Exception:
        util.logexc(LOG, "Consuming user data failed!")
        return (init.datasource, ["Consuming user data failed!"])
    apply_reporting_cfg(init.cfg)

    # Stage 8 - re-read and apply relevant cloud-config to include user-data
	# 实例化Modules类
    # 合并所有cloud-config，包括：/etc/cloud/cloud.cfg{,.d}，/run/cloud-init/cloud.cfg，/proc/cmdline，/var/lib/cloud/instance/cloud-config.txt，/var/lib/cloud/instance/vendor-cloud-config.txt
    mods = stages.Modules(init, extract_fns(args), reporter=args.reporter)
    # Stage 9
    try:
        # 使用mods对象再次重定向日志输出
        outfmt_orig = outfmt
        errfmt_orig = errfmt
        (outfmt, errfmt) = util.get_output_cfg(mods.cfg, name)
        if outfmt_orig != outfmt or errfmt_orig != errfmt:
            LOG.warning("Stdout, stderr changing to (%s, %s)",
                        outfmt, errfmt)
            (outfmt, errfmt) = util.fixup_output(mods.cfg, name)
    except Exception:
        util.logexc(LOG, "Failed to re-adjust output redirection!")
    logging.setupLogging(mods.cfg)

    # give the activated datasource a chance to adjust
    # 调用self.datasource.activate，该方法在user-data和vendor-data渲染后，init_modules执行前调用
    # 写入/var/lib/cloud/instance/obj.pkl
    init.activate_datasource()

    di_report_warn(datasource=init.datasource, cfg=init.cfg)

    # Stage 10
    # 执行init_modules
    return (init.datasource, run_module_section(mods, name, name))
```

### cloud-init modules {#cloud-init-modules}

cmd/main.py, main\_modules\(\)，cloud-init在config和final阶段会运行该函数，分析如下：

```py
def main_modules(action_name, args):
    # config或final
    name = args.mode
    # Cloud-init v. 19.4 running 'modules:config' at Wed, 15 Dec 2021 03:01:15 +0000. Up 280.96 seconds.
    w_msg = welcome_format("%s:%s" % (action_name, name))
    # 实例化Init类
    init = stages.Init(ds_deps=[], reporter=args.reporter)
    # Stage 1
    # 加载配置文件，优先级从低到高为：内置配置 --> /etc/cloud/clouf.cfg{,.d} --> /run/cloud-init/cloud.cfg --> kernel cmdline
    init.read_cfg(extract_fns(args))
    # Stage 2
    try:
    	# 从数据源中获取数据。当obj.pkl缓存文件存在，则从缓存加载数据，
    	# 否则遍历所有数据源，选择能够第一个能够获取数据的数据源当作本实例数据源
        # s.update_metadata([EventType.BOOT_NEW_INSTANCE])，_get_data
        init.fetch(existing="trust")
    except sources.DataSourceNotFoundException:
        # There was no datasource found, theres nothing to do
        msg = ('Can not apply stage %s, no datasource found! Likely bad '
               'things to come!' % name)
        util.logexc(LOG, msg)
        print_exc(msg)
        if not args.force:
            return [(msg)]
    # 如果数据源是从缓存恢复的，且instance-data.json文件缺失，则恢复它
    _maybe_persist_instance_data(init)
    # Stage 3
    # 实例化Modules类
    mods = stages.Modules(init, extract_fns(args), reporter=args.reporter)
    # Stage 4
    # 重定向标准输出到日志文件
    try:
        LOG.debug("Closing stdin")
        util.close_stdin()
        util.fixup_output(mods.cfg, name)
    except Exception:
        util.logexc(LOG, "Failed to setup output redirection!")
    if args.debug:
        # Reset so that all the debug handlers are closed out
        LOG.debug(("Logging being reset, this logger may no"
                   " longer be active shortly"))
        logging.resetLogging()
    logging.setupLogging(mods.cfg)
    apply_reporting_cfg(init.cfg)

    # now that logging is setup and stdout redirected, send welcome
    welcome(name, msg=w_msg)

    # Stage 5
    # 运行各个模块
    return run_module_section(mods, name, name)

```

## 针对redhat系的定制 {#针对redhat系的定制}

收录于centos yum仓库的cloud-init是定制的版本，在开源cloud-init的基础上合入了一系列针对redhat系linux的patch。目前收录如下：

1. Add initial redhat setup。此补丁包含多个补丁，主要包含对默认配置的改动，例如将system\_info.distro更改为rhel，添加默认cloud.cfg配置文件，以及添加一系列systemd服务配置文件
2. Do not write NM\_CONTROLLED=no in generated interface config files。
3. limit permissions on def\_log\_file。添加日志文件用户权限配置选项def\_log\_file\_mode，且设置其默认值为0600
4. sysconfig: Don't write BOOTPROTO=dhcp for ipv6 dhcp。
5. DataSourceAzure.py: use hostnamectl to set hostname。
6. include 'NOZEROCONF=yes' in /etc/sysconfig/network。云上实例需要使用该配置
7. Remove race condition between cloud-init and NetworkManager。移除systemd服务中对NetworkManager的竞争，设置ssh\_deletekeys为1
8. net: exclude OVS internal interfaces in get\_interfaces。
9. Fix requiring device-number on EC2 derivatives。
10. rhel/cloud.cfg: remove ssh\_genkeytypes in settings.py and set in cloud.cfg。在cloud.cfg中添加 ssh\_genkeytypes，首次开机启动时生成公私钥
11. write passwords only to serial console, lock down cloud-init-output.log。
12. ssh-util: allow cloudinit to merge all ssh keys into a custom user file, defined in AuthorizedKeysFile。
13. Stop copying ssh system keys and check folder permissions。
14. Fix home permissions modified by ssh module \(SC-338\)。
15. ssh\_utils.py: ignore when sshd\_config options are not key/value pairs。
16. cc\_ssh.py: fix private key group owner and permissions。

## 针对OpenStack的变更 {#针对openstack的变更}

1. 基于centos8 cloud-init-21.1-9.el8.src.rpm作变更，此包基于开源cloud-init 21.1版本合入了多个redhat系linux定制的patch。
2. cloud-init的detect\_openstack方法（cloudinit\sources\DataSourceOpenStack.py）检测实例是否位于OpenStack，以判断是否应用openstack数据源。由于裸金属实例无法判断是否属于OpenStack，需修改此方法，直接返回true。
3. ...



