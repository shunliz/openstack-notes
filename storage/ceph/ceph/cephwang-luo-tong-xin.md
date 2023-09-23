![](/assets/storage-ceph-asyncmsg1.png)![](/assets/storage-ceph-asynmsg2.png)![](/assets/storage-ceph-asynmsg3.png)



# OSD启动流程

## 初始化过程

初始化的过程在ceph-osd.cc中：

![](https://img-blog.csdnimg.cn/img_convert/d29f2cc8b49bb9b22f23b1e0015f3b46.gif)

```
//创建一个 Messenger 对象，由于 Messenger 是抽象类，不能直接实例化，提供了一个 ::create 的方法来创建子类
Messenger *ms_public = Messenger::create(g_ceph_context, public_msg_type,
                       entity_name_t::OSD(whoami), "client",
                       getpid(),
                       Messenger::HAS_HEAVY_TRAFFIC |
                       Messenger::HAS_MANY_CONNECTIONS);//处理client消息
                       //？？会选择SimpleMessager类实例，SimpleMessager类中会有一个叫做Accepter的成员，会申请该对象，并且初始化。
  Messenger *ms_cluster = Messenger::create(g_ceph_context, cluster_msg_type,
                        entity_name_t::OSD(whoami), "cluster",
                        getpid(),
                        Messenger::HAS_HEAVY_TRAFFIC |
                        Messenger::HAS_MANY_CONNECTIONS);
//下面几个是检测心跳的
  Messenger *ms_hb_back_client = Messenger::create（·····）
  Messenger *ms_hb_front_client = Messenger::create（·····）
  Messenger *ms_hb_back_server = Messenger::create(······）
  Messenger *ms_hb_front_server = Messenger::create（·····）
  Messenger *ms_objecter = Messenger::create（······）
··········

//绑定到固定ip
// 这个ip最终会绑定在Accepter中。然后在Accepter-
>
bind函数中，会对这个ip初始化一个socket，
//并且保存为listen_sd。接着会启动Accepter-
>
start(),这里会启动Accepter的监听线程，这个线
//程做的事情放在Accepter-
>
entry()函数中
/*
messenger::bindv()  --  messenger::bind()  
                      --simpleMessenger::bind()
                            ---- accepter.bind()  
                                   ----创建socket---- socket bind() --- socket listen()
*/
  if (ms_public-
>
bindv(public_addrs) 
<
 0)
    forker.exit(1);
  if (ms_cluster-
>
bindv(cluster_addrs) 
<
 0) 
    forker.exit(1);
···········
//创建dispatcher子类对象
  osd = new OSD(g_ceph_context,
                store,
                whoami,
                ms_cluster,
                ms_public,
                ms_hb_front_client,
                ms_hb_back_client,
                ms_hb_front_server,
                ms_hb_back_server,
                ms_objecter,
                
&
mc,
                data_path,
                journal_path);
·········
// 启动 Reaper 线程
  ms_public-
>
start();
  ms_hb_front_client-
>
start();
  ms_hb_back_client-
>
start();
  ms_hb_front_server-
>
start();
  ms_hb_back_server-
>
start();
  ms_cluster-
>
start();
  ms_objecter-
>
start();

//初始化 OSD模块
     /**
       a). 初始化 OSD 模块
       b). 通过 SimpleMessenger::add_dispatcher_head() 注册自己到
        SimpleMessenger::dispatchers 中, 流程如下:
        Messenger::add_dispatcher_head()
             --
>
 ready()
                 --
>
 dispatch_queue.start()(新 DispatchQueue 线程)
                   --
>
 Accepter::start()(启动start线程)
                     --
>
 accept
                        --
>
 SimpleMessenger::add_accept_pipe
                           --
>
 Pipe::start_reader
                               --
>
 Pipe::reader()
        在 ready() 中: 通过 Messenger::reader(),
        1) DispatchQueue 线程会被启动，用于缓存收到的消息消息
        2) Accepter 线程启动，开始监听新的连接请求.
        */

  // start osd
  err = osd-
>
init();
············
// 进入 mainloop, 等待退出
  ms_public-
>
wait(); // Simplemessenger::wait()
  ms_hb_front_client-
>
wait();
  ms_hb_back_client-
>
wait();
  ms_hb_front_server-
>
wait();
  ms_hb_back_server-
>
wait();
  ms_cluster-
>
wait();
  ms_objecter-
>
wait();
```

消息接收

![](/assets/storage-ceph-asyncmsgrecv.png)消息发送

![](/assets/storage-ceph-asynmsgsend.png)

