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

### 实例分析

```
dig gtm-test.wenteng.site
```

以 gtm-test.wenteng.site 这个域名为例，我们分析一下这个域名解析为 IP 的过程。首先 dig 一下这个域名会得到以下结果。

![](/assets/network-basic-dns5.png)

从上图我们可以看到，gtm-test.wenteng.site 的 DNS 解析过程中其实还包含一次 CNAME 域名的解析（[http://gtm-test.wenteng.site.gtm.volcdns.com](https://link.zhihu.com/?target=http%3A//gtm-test.wenteng.site.gtm.volcdns.com)）。这是因为该域名接入了云调度 GTM，你可以理解为我是在 GTM 平台配置了[http://test.wenteng.site.gtm.volcdns.com](https://link.zhihu.com/?target=http%3A//test.wenteng.site.gtm.volcdns.com)与 IP（76.76.21.61 等）的映射，之后又在云解析 DNS 平台为二级域名 wenteng.site 添加了一条主机记录为 gtm-test，记录值为[http://test.wenteng.site.gtm.volcdns.com](https://link.zhihu.com/?target=http%3A//test.wenteng.site.gtm.volcdns.com)的解析记录。

* 当中国内地用户在浏览器输入访问 gtm-test.wenteng.site 时，本地缓存未命中，会去查询 Local DNS 服务器；
* 若 Local DNS 服务器中没有该域名的缓存记录，则会依次去查询根域 DNS 服务器、顶级域 DNS 服务器、权威 DNS 服务器；
* 根域 DNS 服务器会告知.com 顶级域对应的顶级域解析服务器地址；
* 之后顶级域服务器会告知 wenteng.site 二级域名对应的权威 DNS 服务器地址；
* 最终从权威域名服务器中查询到 gtm-test.wenteng.site 这个域名对应的记录值；
* 查询到的记录类型为 CNAME 类型，非 IP，之后会对这个结果进行又一次的 5-10 过程解析，最终返回 A 类型的 IP 76.76.21.61、76.76.21.22。

实际情况中，因为 Local DNS 服务器有缓存，每一次的查询过程不是一定都要走根域名这个过程的，不然根域 DNS 服务器的流量就太大了。



## DNS 解析的拓展知识

### 普通解析的升级

  


**智能解析**

传统 DNS 解析，不判断访问者来源，而智能 DNS 解析，会判断访问者的来源，为不同的访问者智能返回不同的 IP 地址，可使访问者在访问网站时可获取用户指定的就近 IP 地址，能够减少解析时延，并提升网站访问速度的功效。

  


如果您的业务分布在多个地区的多个 IDC 机房且各机房有独立的 IP 地址，您希望用户根据其所在地区和运营商，接入就近机房，这种场景可以使用智能解析。

  


![](https://pic2.zhimg.com/80/v2-13e759828aef472dc349d2af9aee6969_720w.webp)

  


上图为云解析 DNS 平台 wenteng.site 的解析记录截图，很明显可以看到每个记录都有线路属性。线路是一个二级结构，地区+运营商，DNS 就是根据用户的线路配置和实际请求的来源位置来实现智能解析的。同样是访问 intelligent.wenteng.site 这个域名

* 中国内地 ISP 为电信的用户实际访问到的服务 IP 为 3.3.3.3（默认线路）；
* 中国内地 ISP 为移动的用户实际访问到的服务 IP 为 4.4.4.4（中国移动）；
* 中国内地联通的用户实际访问到的服务 IP 为 1.1.1.1；
* 国外用户实际访问到的服务 IP 为 2.2.2.2。

  


如果你不需要开启智能解析，则直接将所有线路设为默认即可。值得注意的是，当开启智能解析时，默认线路是必须的，不然地区+运营商匹配不到的情况会导致空解析。此外，如果 3.3.3.3 是国外的 IP，中国内地电信用户访问到的服务就是国外的服务，这个可能不符合预期。因此要实现真正的就近接入，解析记录代表的服务器和线路最好是匹配的，不然有可能适得其反。

  


**私网 DNS**

PrivateZone，是基于专有网络 VPC（Virtual Private Cloud）环境的私有 DNS 服务。该服务允许您在自定义的一个或多个 VPC 中将私有域名映射到 IP 地址。通过 PrivateZone，您可以方便地使用私有域名记录来管理 VPC 中的 TOS ECS 等资源，而这些私有域名在 VPC 之外将无法访问。

  


**HTTPDNS**

DNS 存在一些缺点：如 DNS 更新不及时、DNS 劫持问题、流量调度不均衡等。正因为传统 DNS 存在上述问题，因此 HTTPDNS 应运而生。客户端使用 HTTP 协议或 HTTPS 协议发送请求到 HTTPDNS 服务端，从而绕过运营商的 Local DNS，防止域名劫持、实现精准调度。

  


### DNS 负载均衡

  


DNS 负载均衡的本质是多次解析同一个域名可以返回不同的 IP 地址。最简单的实现 DNS 负载均衡的方式就是在 DNS 解析配置平台为一个域名配置多个 IP 并开启负载均衡，实现多次请求的流量打到多个 IP 的目的。

