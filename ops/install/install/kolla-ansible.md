# [MariaDB galera cluster 全部停止后如何再启动](https://www.cnblogs.com/nulige/articles/8470001.html)

### 一、问题场景

1.正式环境下基本上不会出现此类情况

2.测试环境的时候可能会出现，如自己电脑上搞的几个虚拟机上测试，后来全部关机了，再想启动集群，报错了

【系统环境】

CentOS7 + MariaDB10.1.22+galera cluster

【解决方式】

1.正常第一次启动集群，使用命令：galera\_new\_cluster ,其他版本请另行参考

2.整个集群关闭后，再重新启动，则打开任一主机，输入命令：

vim /var/lib/mysql/grastate.dat

```
#GALERA savedd state
version:2.1
uuid: 自己的cluster id
seqno: -1
safe_to_bootstrap:0
```

修改seqno:1

3.重新启动集群命令：galera\_new\_cluster

4.其他节点:systemctl start mariadb



问题二：



```
[root@controller1 haproxy]# galera_new_cluster
Job for mariadb.service failed because the control process exited with error code.
See "systemctl status mariadb.service" and "journalctl -xe" for details.
[root@controller1 haproxy]# tail /var/log/mariadb/mariadb.log
2018-03-21 12:16:18 140168333977920 [Note] WSREP: GCache history reset: f84f94a1-2c38-11e8-8ede-96f87262fb85:0 -> f84f94a1-2c38-11e8-8ede-96f87262fb85:-1
2018-03-21 12:16:18 140168333977920 [Note] WSREP: Assign initial position for certification: -1, protocol version: -1
2018-03-21 12:16:18 140168333977920 [Note] WSREP: wsrep_sst_grab()
2018-03-21 12:16:18 140168333977920 [Note] WSREP: Start replication
2018-03-21 12:16:18 140168333977920 [Note] WSREP: 'wsrep-new-cluster' option used, bootstrapping the cluster
2018-03-21 12:16:18 140168333977920 [Note] WSREP: Setting initial position to 00000000-0000-0000-0000-000000000000:-1
2018-03-21 12:16:18 140168333977920 [ERROR] WSREP: It may not be safe to bootstrap the cluster from this node. It was not the last one to leave the cluster and may not contain all the updates. To force cluster bootstrap with this node, edit the grastate.dat file manually and set safe_to_bootstrap to 1 .
2018-03-21 12:16:18 140168333977920 [ERROR] WSREP: wsrep::connect(gcomm://controller1,controller2,controller3) failed: 7
2018-03-21 12:16:18 140168333977920 [ERROR] Aborting
 
#解决办法
[root@controller1 haproxy]# cat /var/lib/mysql/grastate.dat
# GALERA saved state
version: 2.1
uuid:    d6aea58b-2cbe-11e8-9c9d-b72d8fdd0931
seqno:   -1
safe_to_bootstrap: 0 
 
把safe_to_bootstrap: 0   #修改成safe_to_bootstrap: 1
 
#再启动集群
[root@controller1 haproxy]# galera_new_cluster #其他节点启动服务：systemctl start mariadb
```





二、mysql galera 集群常见问题处理

一、mysql HA集群在断网过久或者所有节点都down了之后的恢复有以下的方法：  
解决方案1：  
1、等三台机器恢复网络通讯后，因为此时的mysql已经异常无法加入集群，因此需要先保证所有的mysql都是down的，再上台执行/usr/libexec/mysqld --wsrep-new-cluster --wsrep-cluster-address='gcomm://' & 这条命令，并进入mysql（只有一台机器能够成功执行，其他机器执行了过几秒钟都会异常退出这个进程，我们这里把能够成功执行的机器称为master）  
2、此时三台只有一台能够成功进入mysql（即执行mysql这条命令），在非master上的两台上一台一台的执行systemctl start mysqld，必须等一台成功了，另一台才能执行。

3、在mysql中执行show status like "wsrep%";结果如下图：

![](https://img-blog.csdn.net/20160918135303863?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

 我们需要保证图中的第一项为synced，以及第二项必须为三个mysql的ip

4、保证3的结果是想要的说明集群已经恢复了，此时需要将master机器上面的/usr/libexec/mysqld --wsrep-new-cluster --wsrep-cluster-address='gcomm://'这个进程kill掉，然后再执行systemctl start mysqld即可

二、mysql HA集群某个节点无故down了并且有一段时间处于down的情况通过以下方式恢复：

1、 若日志里面出现以下日志

160119 14:11:05 \[Warning\] WSREP: Failed to prepare for incremental state transfer: Local state UUID \(00000000-0000-0000-0000-000000000000\) does not match group state UUID \(eb9f50c6-bc95-11e5-a735-9f48e437dc03\): 1 \(Operation not permitted\)

解决方法：删除/var/lib/mysql/grastate.dat 文件（若还存在无法同步的情况则删除galera.cache文件）

2、 若那个down了的节点出现以下日志

（异常情况集群挂了）\[ERROR\] Found 1 prepared transactions! It means that mysqld was not shut down properly last time and critical recovery information \(last binlog or tc.log file\) was manually deleted after a crash. You have to start mysqld with --tc-heuristic-recover switch to commit or rollback pending transactions

解决方法：  
1、/usr/libexec/mysqld start --innodb\_force\_recovery=6  
 1. \(SRV\_FORCE\_IGNORE\_CORRUPT\):忽略检查到的corrupt页。  
  2. \(SRV\_FORCE\_NO\_BACKGROUND\):阻止主线程的运行，如主线程需要执行full purge操作，会导致crash。  
  3. \(SRV\_FORCE\_NO\_TRX\_UNDO\):不执行事务回滚操作。  
  4. \(SRV\_FORCE\_NO\_IBUF\_MERGE\):不执行插入缓冲的合并操作。  
  5. \(SRV\_FORCE\_NO\_UNDO\_LOG\_SCAN\):不查看重做日志，InnoDB存储引擎会将未提交的事务视为已提交。  
  6. \(SRV\_FORCE\_NO\_LOG\_REDO\):不执行前滚的操作。  
如果配置后出现以下情况：  
130507 14:14:01  InnoDB: Waiting for the background threads to start  
130507 14:14:02  InnoDB: Waiting for the background threads to start  
130507 14:14:03  InnoDB: Waiting for the background threads to start  
130507 14:14:04  InnoDB: Waiting for the background threads to start  
130507 14:14:05  InnoDB: Waiting for the background threads to start  
130507 14:14:06  InnoDB: Waiting for the background threads to start  
130507 14:14:07  InnoDB: Waiting for the background threads to start  
130507 14:14:08  InnoDB: Waiting for the background threads to start  
130507 14:14:09  InnoDB: Waiting for the background threads to start  
  
  
需要在galera.cfg中添加这一下：  
如果在设置 innodb\_force\_recovery &gt;2 的同时innodb\_purge\_thread = 0  
2、mysqld --tc-heuristic-recover=ROLLBACK  
3、删除/var/lib/mysql/ib\_logfile\*  
4、当某个mysql节点挂了，并且存在三个mysql所在host有不同的网段，当mysql想重新加入需要一个sst的过程，sst时会需要知道集群中某个节点的ip因此需要制定参数--wsrep-sst-receive-address否则可能出现同步的ip不在三台机器所共有的网段  
解决参考：  
http://blog.itpub.net/22664653/viewspace-1441389/  
  
  
三、一个mysql节点若down了一段时间。重新启动的时候需要一些时间去同步数据，服务的启动超时时间不够，导致服务无法启动，解决方法如下：  
The correct way to adjust systemd settings so they don't get overwritten is to create a directory and file as such:  
/etc/systemd/system/mariadb.service.d/timeout.conf  
\[Service\]  
  
TimeoutStartSec=12min  
  
  
或者直接修改/usr/lib/systemd/system/mariadb.service  
\[Service\]  
  
TimeoutStartSec=12min  
这里的时间最少要大于90s，默认是90s之后执行 systemctl daemon-reload再重启服务即可  
四、日志中出现类似如下错误：  
160428 13:54:49 \[ERROR\] Slave SQL: Error 'Table 'manage\_operations' already exists' on query. Default database: 'horizon'. Query: 'CREATE TABLE \`manage\_operations\` \(  
    \`id\` integer AUTO\_INCREMENT NOT NULL PRIMARY KEY,  
    \`name\` varchar\(50\) NOT NULL,  
    \`type\` varchar\(20\) NOT NULL,  
    \`operation\` varchar\(20\) NOT NULL,  
    \`status\` varchar\(20\) NOT NULL,  
    \`time\` date NOT NULL,  
    \`operator\` varchar\(50\) NOT NULL  
\) default charset=utf8', Error\_code: 1050  
160428 13:54:49 \[Warning\] WSREP: RBR event 1 Query apply warning: 1, 28585  
160428 13:54:49 \[Warning\] WSREP: Ignoring error for TO isolated action: source: 752eecd1-0ce0-11e6-83fc-3e0502d0bdd2 version: 3 local: 0 state: APPLYING flags: 65 conn\_id: 24053 trx\_id: -1 seqnos \(l: 28668, g: 28585, s: 28584, d: 28584, ts: 80224119986850\)  
导致进程异常关闭，  
此时可以通过执行mysqladmin flush-tables来刷新表项，这个问题的原因是三个节点之间的表同步存在问题，刷新一下表即可  
  
  
五、日志出现以下错误：  
160520 10:48:23 \[Note\] WSREP: COMMIT failed, MDL released: 367194  
160520 10:48:23 \[Note\] WSREP: cert failure, thd: 358780 is\_AC: 0, retry: 0 - 1 SQL: commit  
160520 10:48:23 \[Note\] WSREP: cert failure, thd: 358784 is\_AC: 0, retry: 0 - 1 SQL: commit  
160520 10:48:23 \[Note\] WSREP: COMMIT failed, MDL released: 367188  
160520 10:48:23 \[Note\] WSREP: cert failure, thd: 359683 is\_AC: 0, retry: 0 - 1 SQL: commit  
160520 10:48:23 \[Note\] WSREP: cert failure, thd: 358808 is\_AC: 0, retry: 0 - 1 SQL: commit  
160520 10:48:23 \[Note\] WSREP: cert failure, thd: 367191 is\_AC: 0, retry: 0 - 1 SQL: commit  
160520 10:48:23 \[Note\] WSREP: cert failure, thd: 367196 is\_AC: 0, retry: 0 - 1 SQL: commit  
160520 10:48:23 \[Note\] WSREP: cert failure, thd: 367194 is\_AC: 0, retry: 0 - 1 SQL: commit

160520 10:48:23 \[Note\] WSREP: cert failure, thd: 367188 is\_AC: 0, retry: 0 - 1 SQL: commit

8、日志出现以下错误：

160820  3:13:41 \[ERROR\] Error in accept: Too many open files  
160820  3:19:42 \[ERROR\] Error in accept: Too many open files  
160827  3:16:24 \[ERROR\] Error in accept: Too many open files  
160831 17:20:52 \[ERROR\] Error in accept: Too many open files  
160831 19:54:29 \[ERROR\] Error in accept: Too many open files  
160831 20:21:53 \[ERROR\] Error in accept: Too many open files  
160901 11:25:57 \[ERROR\] Error in accept: Too many open files

解决方法

vim /usr/lib/systemd/system/mariadb.service

 \[Service\]  
 LimitNOFILE=10000

默认的mysql的open\_file\_limits是1024将该项增大，并且修改vim /etc/my.cnf.d/server.cnf该文件的open\_files\_limit值

systemctl daemon-reload

systemctl restart mysqld

查看mysql的open\_file\_limits值是否调整成功

cat /proc/$pid/limit

其中$pid为mysql进程的pid看看值是否调整成功，并看看日志是否还会出现上述错误。

