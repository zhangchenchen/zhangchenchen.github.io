---
layout: post
title:  "openstack系列 --关于keystone配置详解"
date:   2016-10-24 15:01:55
tags: openstack
---


## 目录结构


[keystone 配置文件综述 ](#A)

[keystone 配置文件详解](#B)



<a name="A"></a>

## keystone 配置文件综述

关于keystone的概述以及整体架构我们已经在之前的[一篇文章](http://zhangchenchen.github.io/2016/09/16/openstack-keystone/)中做过介绍，不再赘述。这里具体介绍下keystone的配置方面的知识。

首先，keystone的配置文件主要有两个，一个是通常位于/etc/keystone/或/etc/目录下的keystone.conf，另一个是位于keystone安装根目录下的keystone-paste.ini。其中，后者其实就是paste deploy 文件，我们这里不需要关注，主要讨论keystone.conf文件。

tips:keystone默认端口是35357，因此我们最好将35357端口从临时端口范围中删除，以免该端口被用作临时端口。
命令如下： 
>sysctl -w 'net.ipv4.ip_local_reserved_ports=35357'

如果是想重启后仍然有效，在/etc/sysctl.conf 或/etc/sysctl.d/keystone.conf文件后追加： net.ipv4.ip_local_reserved_ports = 35357 


<a name="B"></a>

## keystone 配置文件详解

keystone.conf文件中所有的配置项如下：

![keystone-config](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/20161024-keystone-config.png)

接下来我们从其中挑选几个值得注意的配置项详细说明下：

1. 关于TOKEN

对于token的持久化有两种形式：一种是以key/value对的形式存储，另一种是存储在SQL数据库中。
需要在[token] 配置项中的driver选项进行具体配置。

token provider 大致有以下四种形式：

 - UUID:长度固定为 32 Byte 的随机字符串,UUID token 简单美观,验证流程如下：
 
![uuid](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/20161024-uuid.png)
由于每当 OpenStack API 收到用户请求，都需要向 Keystone 验证该 token 是否有效。随着集群规模的扩大，Keystone 需处理大量验证 token 的请求，在高并发下容易出现性能问题。为了杜绝keystone成为瓶颈，引出了下面的几种。

 - PKI:验证流程如下：
![pki](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/20161024-pki.png)
可以看出，PKI token携带更多用户信息的同时还附上了数字签名，以支持本地认证，从而避免了步骤 4。因为 PKI token 携带了更多的信息，这些信息就包括 service catalog，随着 OpenStack 的 Region 数增多，service catalog 携带的 endpoint 数量越多，PKI token 也相应增大，很容易超出 HTTP Server 允许的最大 HTTP Header(默认为 8 KB)，导致 HTTP 请求失败。
 - PKIZ:顾名思义，就是对PKI token进行压缩，但压缩效果有限，无法良好的处理 token size 过大问题。原理同上，无须赘述。
 - FERNET:前三种 token 都会持久性存于数据库，与日俱增积累的大量 token 引起数据库性能下降，所以用户需经常清理数据库的 token（ 命令为：keystone-manage token_flush）。为了避免该问题，社区提出了 Fernet token，它携带了少量的用户信息，大小约为 255 Byte，采用了对称加密，无需存于数据库中。

![fernet](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/20161024-fernet.png)

 - 几种token provider比较如下：
 
![token比较](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/20161024-token%20compare.png)

2. Caching Layer

启动keystone的的cache功能，首先要在[cache]配置项，设置为enabled,指定backend等等。然后再在其他具体配置项进行设置。支持cache的地方有以下几个地方：
 - token
 - resource
 - role
具体配置根据backend的不同也不尽相同。

3. Service Catalog
keystone 提供了两种配置选项。
 - SQL-based Service Catalog：配置选项如下：

```
  [catalog]
  driver = sql 
```
可以通过查阅以下命令进行相关操作：

```bash
 openstack --help
 openstack help service create
 openstack help endpoint create
```

 - File-based Service Catalog (templated.Catalog)：就是基于一个配置模板文件，配置选项如下：
  
  ```
  [catalog]
  driver = templated
  template_file = /opt/stack/keystone/etc/default_catalog.templates
  ```

  模板文件示例：
  ![template](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/20161024-template.png)


4. 其他

其实，除了sql数据库，我们也可以把数据存在文件中，以 LDAP形式，不过用的不多，不再赘述。




## 参考文章

[openstack doc ](http://docs.openstack.org/developer/keystone/configuration.html)

[理解 Keystone 的四种 Token](http://www.openstack.cn/?p=5120)




***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***

