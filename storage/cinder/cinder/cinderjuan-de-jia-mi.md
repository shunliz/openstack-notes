目标：

创建并使用加密的cinder卷

准备:

-openstack

![](/assets/storage-cinder-encryptcidercodearch.png)

过程：

修改配置文件：

将fixed\_key的值设置为cinder-volume-key和一组十六位的十六进制的密钥（控制节点和计算节点）

\[root@compute ~\]\# ssh controller

root@controller's password:

Last login: Tue May 28 12:10:00 2019 from compute

\[root@controller ~\]\# sed -n 3524,3532p /etc/nova/nova.conf

\[keymgr\]

\#

\# From nova

\#

\# Fixed key returned by key manager, specified in hex \(string value\)

fixed\_key=cinder-volume-key

fixed\_key=0000000000000000

\[root@controller ~\]\# sed -n 2794,2811p /etc/cinder/cinder.conf

\[KEYMGR\]

\#

\# From cinder

\#

\# Authentication url for encryption service. \(string value\)

\#encryption\_auth\_url = [http://localhost:5000/v3](http://localhost:5000/v3)

\# Url for encryption service. \(string value\)

\#encryption\_api\_url = [http://localhost:9311/v1](http://localhost:9311/v1)

\# The full class name of the key manager API class \(string value\)

\#api\_class = cinder.keymgr.conf\_key\_mgr.ConfKeyManager

\# Fixed key returned by key manager, specified in hex \(string value\)

fixed\_key = cinder-volume-key

fixed\_key = 0000000000000000

重启服务：

\[root@controller ~\]\# systemctl restart openstack-nova-compute

\[root@controller ~\]\# systemctl restart openstack-cinder-volume

\[root@compute ~\]\# systemctl restart openstack-nova-compute

\[root@compute ~\]\# systemctl restart openstack-cinder-volume

创建加密卷类型：

\[root@controller ~\]\# cinder type-create luks

+--------------------------------------+------+-------------+-----------+

\|                  ID                  \| Name \| Description \| Is\_Public \|

+--------------------------------------+------+-------------+-----------+

\| bd278099-ad80-4f54-b1da-e0734ee33cba \| luks \|      -      \|    True   \|

+--------------------------------------+------+-------------+-----------+

\[root@controller ~\]\# cinder encryption-type-create --cipher aes-xts-plain64 --key\_size 512 --control\_location front-end luks nova.volume.encryptors.luks.LuksEncryptor

+--------------------------------------+-------------------------------------------+-----------------+----------+------------------+

\|            Volume Type ID            \|                  Provider                 \|      Cipher     \| Key Size \| Control Location \|

+--------------------------------------+-------------------------------------------+-----------------+----------+------------------+

\| bd278099-ad80-4f54-b1da-e0734ee33cba \| nova.volume.encryptors.luks.LuksEncryptor \| aes-xts-plain64 \|   512    \|    front-end     \|

+--------------------------------------+-------------------------------------------+-----------------+----------+------------------+

创建cinder卷：

未加密：\[root@controller ~\]\# cinder create --name unencrypted 1

加密：\[root@controller ~\]\# cinder create --name encrypted --volume-type luks 1

挂载到云主机进行测试：

加密卷：

\[root@controller ~\]\# nova volume-attach iaas 89f2cc72-435e-4136-b32f-44acdd96a8a7

+----------+--------------------------------------+

\| Property \| Value                                \|

+----------+--------------------------------------+

\| device   \| /dev/vdb                             \|

\| id       \| 89f2cc72-435e-4136-b32f-44acdd96a8a7 \|

\| serverId \| 3089a676-1394-4e0b-bad7-e9ed1f43e676 \|

\| volumeId \| 89f2cc72-435e-4136-b32f-44acdd96a8a7 \|

+----------+--------------------------------------+

未加密：

\[root@controller ~\]\# nova volume-attach iaas c4fa8087-e648-4ee4-ba24-be8d56835d98

+----------+--------------------------------------+

\| Property \| Value                                \|

+----------+--------------------------------------+

\| device   \| /dev/vdc                             \|

\| id       \| c4fa8087-e648-4ee4-ba24-be8d56835d98 \|

\| serverId \| 3089a676-1394-4e0b-bad7-e9ed1f43e676 \|

\| volumeId \| c4fa8087-e648-4ee4-ba24-be8d56835d98 \|

+----------+--------------------------------------+

测试：

到云主机里向卷内写入数据

\[root@xiandian ~\]\# echo "encrypted /dev/vdb Hello,world" &gt;&gt; /dev/vdb

\[root@xiandian ~\]\# echo "unencrypted /dev/vdc Hello,world" &gt;&gt; /dev/vdc

到cinder卷所在节点验证：

未加密的卷信息就会暴露出来

\[root@compute ~\]\# strings /dev/mapper/cinder--volumes-volume--\* \| grep "Hello,world"

unencrypted /dev/vdc Hello,world

