---
layout: post
title:  "Openstack-- OpenStack 与OpenNebula的前世今生"
date:   2017-09-12 16:25:30
tags: 
  - openstack
---


## 引出

本篇博文说实话感觉滞后太多，因为关于云计算开源框架的争论在前几年还甚嚣尘上，自从openstack风卷残云之势席卷整个云计算圈之后，之前的一些老前辈貌似都默默退出了，这也包括本文的一个主角OpenNebula。其实OpenNebula一直都在，只不过OpenStack的名气太大，以至于其他框架都被无意间忽略了。不过随着OpenStack项目的飞速膨胀，复杂度与可维护性也随之疯涨，OpenNebula这个一直秉承“Simplicity,Flexibility”的框架也得到了不少人的拥趸，本文就主要讲讲这个本应在几年前讨论的话题：OpenStack 与OpenNebula，孰优孰劣。

![openstack vs opennebula](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/2017-09-12-opennebula-vs-openstack.jpg)

## OpenStack VS OpenNebula

### 定位的区别

Openstack从一开始就是打算跟AWS正面刚，所以一直借鉴（抄袭）AWS,可以认为是走公有云（提供基础设施）路线的，而OpenNebula则是走企业云（数据中心虚拟化）路线的，是打算跟vmware干仗的。但随着各个厂商的站队，OpenStack突然红了，但是红了也是有代价的，OpenStack的发展方向开始被Foundation控制，而这个Foundation是由各大厂商（以red hat为首）控制的，所以OpenStack其实是面向这些巨头们的，当然其中不乏各大公司的博弈，君不见早期launchpad上多个顽疾似的bug一直无人问津，而各大厂商的适配driver却早已开发完善。OpenNebula一直有一个核心开发组织掌握着开发的整体方向，相对于OpenStack,"铜臭味"没那么重，但也因为没有巨头的摇旗呐喊，所以开发进度不是很快。
所以，按照OpenNebula开发者的话说：Openstack是服务于foundation的，而OpenNebula才是真正服务于用户的。

### 整体架构的区别

这个区别对于技术人员来说就更感兴趣了。因为刚开始的定位不同，自然架构也是天壤之别，先上两张图直观感受下：

![openstack-arch](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/20170912openstack-arch.jpg)

![opennebula](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/20170912-opennebula-arch.jpg)

一张图就可以大致了解到，如今的OpenStack俨然是一个庞然大物，不过好在各个子项目分工明确，划分成层之后如下图（图比较老了，有些项目已经毕业了）：

![openstack-arch](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/20170913-openstack-layer.jpg)

再看下具体部署情况：

openstack的典型部署架构：
![openstack-deploy](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/20170913085313-openstack-deploy.jpg)

opennebula的典型部署架构：
![opennebula-deploy](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/20170913085402-open-nebula-deploy.jpg)

可以看出，在计算节点openstack要部署很多agent(包括nova-compute)，而opennebula只要保证可以SSH连接以及有hypervisor就可以，是一种无侵入式的设计。


### 支持服务的区别

网上一张截图（时效性不保证）：

![opennebula-vs-openstack](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/20170913090037-feature.jpg)




### 总结 

Openstack因为各大厂商力推的原因，走的是大而全的路线，你能想到的云计算服务，openstack基本都有对应的子项目，但也造成了架构复杂，学习路线陡峭，后期维护成本高等现象。OpenNebula走的是小而美的路线，无侵入式设计，架构简单，容易上手，但是因为保证无侵入式，所以要大量调用shell，动用的编程语言包括C++,ruby，shell等，项目代码相对混乱，代码可读性不高。
综上所述，如果是建一个求稳定性，且物理环境一致，规模相对较大的私有云，Openstack是首选，而如果计算节点系统不一致，特别是计算节点有公有云的情况(也就是搭建混合云)，可以考虑OpenNebula。



## 参考文章


[OpenNebula vs. OpenStack: User Needs vs. Vendor Driven ](https://opennebula.org/opennebula-vs-openstack-user-needs-vs-vendor-driven/)

[Comparing OpenNebula and OpenStack: Two Different Views on the Cloud](https://opennebula.org/comparing-opennebula-and-openstack-two-different-views-on-the-cloud/)

[OpenNebula 4.14 Hands-on Tutorial](https://www.slideshare.net/opennebula/opennebula-414-handson-tutorial?ref=https://opennebula.org/documentation/tutorials/)

***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***