## external-provisioner源码分析（2）-main方法与Leader选举分析

关联链接

external-provisioner组件的源码分析分为三部分：

（1）主体处理逻辑分析；

（2）main方法与Leader选举分析；

（3）组件启动参数分析。



1.main方法分析

主要对main方法的主要逻辑进行分析，以及分析下组件的EventHandler，看该组件list/watch哪些对象，对象事件来了怎么处理，以及claimQueue与volumeQueue的对象来源。



main方法主要逻辑分析

main方法主要逻辑：

（1）解析启动参数；

（2）根据配置建立clientset；

（3）与csi driver建立连接，建立grpc client；

（4）进行grpc探测（探测csi driver服务是否准备好），直至探测成功；

（5）与csi driver进行通信，获取driver名称与能力；

（6）根据clientset建立informers；

（7）构建provisionController对象；

（8）定义run方法（包括了provisionController.Run）；

（9）根据--enable-leader-election组件启动参数配置决定是否开启Leader 选举，当不开启时，直接运行run方法，开启时调用le.Run\(\)。





