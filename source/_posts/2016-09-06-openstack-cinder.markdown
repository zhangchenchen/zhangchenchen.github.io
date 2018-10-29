---
layout: post
title:  "openstack 系列 --cinder"
date:   2016-09-06 15:01:55
tags: openstack
---


## 目录结构

在学习一个新东西的时候，我们其实默认是带着问题去学习的。但当我们真正在卷帙繁多的互联网或者书籍里获取信息时。之前的问题往往就淡化了，反而更多的是跟着资料所给的思路去学习。这样有好处也有坏处，好处是我们不用费心去思考，而坏处也是如此，不用思考带来的后果就是这些信息虽然当时我们理解了，但很难在我们的脑海中建立知识图谱，比较容易遗忘。

所以以后我会在写文章的时候先把问题具象化，从自己问题的角度出发去寻找答案.

本篇我们的主题是openstack的块存储项目cinder,那么第一个问题自然是 *cinder是干什么用的？* 由这个问题出发，我们逐渐引出 *cinder如何实现？* ，*cinder的具体功能* 。

[cinder是干什么用的 ](#A)

[cinder如何实现 ](#B)

[cinder的具体功能](#C)


<a name="A"></a>

## cinder是干什么用的

openstack中创建虚拟机的时候是附带一块硬盘的，但这块硬盘随着虚拟机的destroy也相应的删除了。所以，openstack推出了两种应付持久存储的解决方案，一个是之前我们提过的对象存储swift，一个就是现在我们讲的块存储cinder。
块存储,对象存储的概念我们[之前](http://zhangchenchen.github.io/2016/09/02/openstack-swift/)已介绍过，不再赘述。
![存储对比](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/20160907openstack-storage-compare.jpg)
需要说明的是，cinder并不是新开发的一个块设备存储系统，它更像是一个资源管理系统，对不同的存储后端进行封装，以统一的API形式向虚拟机提供持久块存储资源。对于不同的存储后端，它采用插件的形式，结合不同的后端存储的驱动提供块存储服务。

<a name="B"></a>

## cinder如何实现

首先看一下cinder的主要组件：
![cinder architecture](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/20160907-openstack-cinder-part.jpg)
简单介绍下这几个组件：
 
 - cinder-api: 负责接受和处理外界的API请求，并将请求放入RabbitMQ队列，交由其他程序执行。
 
 - cinder-scheduler: 在多个存储节点的情况下，新建或迁移volume的时候由该程序依据指定的算法选出合适的节点。算法有过滤算法和权重算法两种。
 
 - cinder-volume： 运行在存储节点上，管理存储空间，处理cinder数据库维护状态的读写请求，通过消息队列和直接在块存储设备或软件上与其他进程交互。每个存储节点都有一个Volume Service，若干个这样的存储节点联合起来可以构成一个存储资源池。cinder-volume为后端的volume provider 定义了统一的 driver 接口，volume provider 只需要实现这些接口，就可以 driver 的形式即插即用到 OpenStack 中。

接下来介绍下这种插件机制以及后端的具体存储：
后端存储大致可以分为两类，一类是软件实现的存储系统，比如LVM,ceph,sheepdog等。另一类是商用的硬件支持的专业存储系统，比如IBM,HP,EMC等公司的专业存储系统。

![不同存储](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/20160907openstack-cinder-storage.jpg)

以LVM 为例：

![lvm](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/20160907-cinder-lvm.jpg)

此处简单介绍下iSCSI协议,引自 [wiki](https://zh.wikipedia.org/wiki/ISCSI):

 > iSCSI利用了TCP/IP的port 860 和 3260 作为沟通的渠道。透过两部计算机之间利用iSCSI的协议来交换SCSI命令，让计算机可以透过高速的局域网集线来把SAN模拟成为本地的储存装置。iSCSI使用 TCP/IP 协议（一般使用TCP端口860和3260）。 本质上，iSCSI 让两个主机通过 IP 网络相互协商然后交换 SCSI 命令。这样一来，iSCSI 就是用广域网仿真了一个常用的高性能本地存储总线，从而创建了一个存储局域网（SAN）。不像某些 SAN 协议，iSCSI 不需要专用的电缆；它可以在已有的交换和 IP 基础架构上运行。然而，如果不使用专用的网络或者子网（ LAN 或者 VLAN ），iSCSI SAN 的部署性能可能会严重下降。于是，iSCSI 常常被认为是光纤通道（Fiber Channel）的一个低成本替代方法，而光纤通道是需要专用的基础架构的。但是，基于以太网的光纤通道（FCoE）则不需要专用的基础架构。



其他商用存储系统：

![others](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/20160907-cinder-other%20-storage.jpg)

两者对比，[check this!](chrome-extension://ikhdkkncnoglghljlkmcimlnlhkeamad/pdf-viewer/web/viewer.html?file=https%3A%2F%2Fevents.linuxfoundation.org%2Fsites%2Fevents%2Ffiles%2Fslides%2FCloudOpenJapan2014-Kimura_0.pdf)


<a name="C"></a>

## cinder的具体功能

![cinder functure](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/20160907-cinder-functure.jpg)

从代码层面了解具体实现的流程，[check this!](https://www.ibm.com/developerworks/community/blogs/132cfa78-44b0-4376-85d0-d3096cd30d3f/entry/Create_Volume_%E6%93%8D%E4%BD%9C_Part_III_%E6%AF%8F%E5%A4%A95%E5%88%86%E9%92%9F%E7%8E%A9%E8%BD%AC_OpenStack_52?lang=en)





## 参考文章

[探索 OpenStack 之（9）：深入块存储服务Cinder ](http://www.cnblogs.com/sammyliu/p/4219974.html)

[每天五分钟玩转openstack ](https://www.ibm.com/developerworks/community/blogs/132cfa78-44b0-4376-85d0-d3096cd30d3f/entry/%E7%90%86%E8%A7%A3_Cinder_%E6%9E%B6%E6%9E%84_%E6%AF%8F%E5%A4%A95%E5%88%86%E9%92%9F%E7%8E%A9%E8%BD%AC_OpenStack_45?lang=en)

[openstack 设计与实现](https://book.douban.com/subject/26374647/)

***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***

