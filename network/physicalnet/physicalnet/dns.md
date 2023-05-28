### 域名的构成

  


下图为域名的结构。其实每个域名都是有根域的，如[http://www.volcengine.com](https://link.zhihu.com/?target=http%3A//www.volcengine.com)其实应该是 www.volcengine.com.，域名末尾的点就是根域名，很多情况下根域名是可以省略掉的。在上述例子中，com 为顶级域名，[http://volcengine.com](https://link.zhihu.com/?target=http%3A//volcengine.com)是二级域名或主域名，[http://www.volcengine.com](https://link.zhihu.com/?target=http%3A//www.volcengine.com)是子域名或分域名。值得注意的是，顶级域名不一定只由一个域名构成，也可以由两个域名构成。虽然.com、.cn 都是顶级域名，但是.[http://com.cn](https://link.zhihu.com/?target=http%3A//com.cn)也是顶级域名。

![](/assets/network-basic-dns1.png)

## 解析记录的类型

![](/assets/network-basic-dns3.png)

## DNS 解析的过程

![](/assets/network-basic-dns2.png)

DNS 解析的过程可以分为本地查询\(1、2）与线上查询（3-11）。

  


**本地查询**

本地查询可以分为 host 文件查询与本地缓存查询。当用户在浏览器中访问域名时，会先进行本地查询，若本地查询命中，则直接返回；未命中，则需要访问线上的 DNS 服务器进行解析。

  


**线上查询**

线上 DNS 解析主要包含：Local DNS 服务器、根域 DNS 服务器、顶级域 DNS 服务器、权威域 DNS 服务器。Local DNS 服务器不在客户端本地，一般为运营商提供的线上 DNS 服务器；权威 DNS 是特定域名记录在域名注册商处所设置的 DNS 服务器，用于特定域名本身的管理。

  


线上查询主要分为递归查询和迭代查询：递归查询是浏览器把任务交给 Local DNS ，然后等待 Local DNS 返回结果；迭代查询是 Local DNS 分别向各级 DNS 服务器发送查询请求，直到获取 DNS 解析结果为止。

  


总的来说，客户端向 Local DNS 服务器进行查询，如果命中 Local DNS 服务器的缓存，直接返回 IP；没有命中，Local DNS 服务器会向各级域名服务器进行查询，直到最终解析出 IP 为止。

