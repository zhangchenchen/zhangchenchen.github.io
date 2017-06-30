---
layout: post
title:  "ceph系列--ceph存储介绍"
date:   2016-09-29 14:24:25
tags: ceph
---

## 目录结构：


[关于分布式存储](#A)

[ceph 是什么 ](#B)

[ceph 架构](#B)

[what makes ceph unique](#C)






<a name="A"></a>

## 关于分布式存储

在了解ceph之前，我们最好还是先了解一下分布式存储的一些概念。分布式存储是随着存储容量的需求变大而出现的概念。由于存储需求变大，出现了两种解决方案，一种是scale up,采用容量更大，价格更昂贵的存储机器。

![scale-up](http://7xrnwq.com1.z0.glb.clouddn.com/scale-up.png) 

另一种是scale out,利用众多的普通存储机器（PC机）横向扩展成一个大的存储集群。

![scale-out](http://7xrnwq.com1.z0.glb.clouddn.com/20160929scale-out.png)

第二种就是分布式存储的概念了。相对采用昂贵的专用商业存储设备来说，这种存储的性价比更高，但也因此引出了分布式存储几个比较棘手的问题，比如数据如何组织存储，如何保证高可用性，如何保证冗余数据的一致性，是否支持分布式事务，如何避免单点失败等等，这些概念可以参考这两篇文章，[分布式存储系统 知识体系](http://wuchong.me/blog/2014/08/07/distributed-storage-system-knowledge/),[An Introduction to Distributed Systems](http://webdam.inria.fr/Jorge/html/wdmch15.html)。





<a name="B"></a>

## ceph 是什么

以下篇目是基于这场presentation，[Ceph Intro & Architectural Overview](https://www.youtube.com/watch?v=7I9uxoEhUdY).

我们首先看一下ceph 的设计哲学：

![ceph design](http://7xrnwq.com1.z0.glb.clouddn.com/20160929-ceph-design.png)

由此我们也可以看出ceph的一些特点：开源，社区驱动，可扩展，无单点故障，软件层面，自我管理。

那具体提供什么服务呢？看下图

![ceph service](http://7xrnwq.com1.z0.glb.clouddn.com/20160929-ceph-service.png)

对应过来就是对象存储，块存储，文件系统存储。



<a name="C"></a>

## ceph 架构


再从技术层面看下ceph架构

![ceph-architect](http://7xrnwq.com1.z0.glb.clouddn.com/2016-09-29ceph-architect.png)

我们从底往上一层层的介绍，首先是RADOS,我们从上图的介绍中也可以看出，RADOS是一系列的节点，这些节点上跑着两种程序，跑着OSD程序的我们称为OSD节点（storage node），跑着MON程序的我们称为MON节点（monitor node）。

![rados-1](http://7xrnwq.com1.z0.glb.clouddn.com/2016-09-29-rados-1.png)

我们再详细看下OSD节点。

![rados-osd](http://7xrnwq.com1.z0.glb.clouddn.com/20160929-rados-osd.png)

每个OSD节点内，有多个磁盘disk,这些磁盘上是对应的文件系统（官网推荐 xfs），在往上就是我们的逻辑存储单元OSD，每个磁盘对应一个OSD。我们的数据就是存储在OSD里。
OSD与MON节点各自功能如下：

![OSD VS MON](http://7xrnwq.com1.z0.glb.clouddn.com/20160929-OSD-MON.png)

可以看出MON节点只负责集群状态的维护，具体存储交给OSD处理。

再看下LIBRADOS的作用。

![ceph-librados-service](http://7xrnwq.com1.z0.glb.clouddn.com/20160929ceph-librados-service.png)

简而言之，就是为Application的调用提供了多种语言版本的接口。通信机制为SOCKET。

![librados](http://7xrnwq.com1.z0.glb.clouddn.com/20160929-librados.png)

再往上是RADOSGW.

![RADOSGW-1](http://7xrnwq.com1.z0.glb.clouddn.com/20160929-radosgw-1.png)

![RADOSGW-1](http://7xrnwq.com1.z0.glb.clouddn.com/20160929-radosgw-2.png)

简而言之，就是在LIBRADOS提供的API的基础上进行封装对外提供对象存储的REST服务。

与之并列的是RBD，也就是ceph提供的块存储服务。我们最常用的就是attach到某台虚拟机上（如下图），Ross Turk还提到了另外两种用法，感兴趣可以去看一下。

![ceph-rbd](http://7xrnwq.com1.z0.glb.clouddn.com/20160929ceph-rbd-1.png)

![ceph-rbd](http://7xrnwq.com1.z0.glb.clouddn.com/20160929-ceph-rbd-2.png)

关于ceph提供的文件系统因为用的不多，不再详述。



<a name="D"></a>

## what makes ceph unique

第一个就是crush算法，一种计算寻址而非查找寻址的方法。

![crush-3](http://7xrnwq.com1.z0.glb.clouddn.com/20160929-crush-3.png)

 图中的placement group 有很重要的作用，有了它，我们底层添加节点或节点崩溃，ceph都会自动更新，而对用户是透明的。
![crush-1](http://7xrnwq.com1.z0.glb.clouddn.com/20160929-crush-1.png)

![crush-2](http://7xrnwq.com1.z0.glb.clouddn.com/20160929-crush-2.png)

其次是 rbdlayering ,参考这里[RBD LAYERING](http://docs.ceph.com/docs/master/dev/rbd-layering/)



## 参考文章

[Ceph Intro & Architectural Overview](https://www.youtube.com/watch?v=7I9uxoEhUdY)

[Ceph Intro and Architectural Overview by Ross Turk](http://www.slideshare.net/buildacloud/ceph-intro-and-architectural-overview-by-ross-turk)

[理解 OpenStack + Ceph （2）：Ceph 的物理和逻辑结构 [Ceph Architecture]](http://www.cnblogs.com/sammyliu/p/4836014.html)

[ceph doc](http://docs.ceph.com/docs/master/)
***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***
