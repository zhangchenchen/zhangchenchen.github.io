---
layout: post
title:  "Kuberbetes-- kubernetes网络方案总结（一）"
date:   2017-12-12 12:03:13
tags: 
  - Kubernetes
---


## 概述

kubernetes本身是不提供网络方案的，我们需要通过addon的方式为k8s集群提供网络功能。目前有很多这种第三方的解决方案，从实现原理上来说大致分为两种，一种是Overlay Networking，也就是通过封包解包的方式构造一个隧道，这种方案在IAAS层用的比较多，比如openstack-neutron就使用的ovs，这种方式因为封包解包的动作，所以性能会有所下降，调试也比较复杂（搞openstack的深有体会）。不过部署简单，架构清晰。代表性的方案有flannel，weave，calico(ipip)，openvswitch等。还有一种方案就是通过路由来实现(更改iptables等手段)，这种方式性能损耗少，不过随着集群内容器增加，路由记录也会不断增加，性能随之逐渐下降。这种方案的典型代表是calico(bgp)，Macvlan（需特殊二层路由器），contiv等。

下面就说下几种网络方案的对比，并在下一篇博客中详细叙述下两种最常见的网络方案：flannel和calico。

## 几种典型网络方案对比

博主没有条件对这几种网络方案进行具体实践对比，从网上搜了一些比较靠谱的测评，罗列如下。

### 性能对比

性能对比，博主找到了两篇相对靠谱的文章，一篇是从hacker news上面找到的，后来发现了[中文翻译版](http://dockone.io/article/1115)，这篇文章主要是对比了Docker host方案，flannel，IPVlan三种方案，这里提一嘴IPVlan，它是 Linux 内核中的一个驱动，有了它，无需使用桥接接口，就可以创建拥有唯一 IP 地址的虚拟网络接口。IPvlan 是一个相对较新的解决方案，现在还没有能完成上述过程的自动化工具。如果集群的机器和容器比较多，部署 IPvlan 的难度不小，不容易设置。直接上这篇文章的结论吧：

![test-result](http://7xrnwq.com1.z0.glb.clouddn.com/2017121-test-result.jpg)
总结一下就是私有云环境下flannel + host-gw方案顶呱呱。

第二篇文章是来自有容云团队的，他们覆盖的方案范围比较广，详情可参考[Kubernetes网络原理及方案](http://www.youruncloud.com/blog/131.html)，这里直接贴测试结果：

![yourong-test-result-1](http://7xrnwq.com1.z0.glb.clouddn.com/20171211-test-result-you-1.jpg)

![yourong-test-result-2](http://7xrnwq.com1.z0.glb.clouddn.com/20171211-test-result-you-2.jpg)

再次说一下，具体的网络方案还是得看实际应用场景，是私有云环境，公有云环境，还是混合云环境，有无SDN，节点数量，集群规模等等，以上测试仅供参考。


### 特点对比

没想到搜到了一篇学长的[博客](http://chunqi.li/2015/11/15/Battlefield-Calico-Flannel-Weave-and-Docker-Overlay-Network/#Conclusion)，翻了一下他的linkedin，立马汗颜，扯远了，文章写的很详细，省的自己整理了，还是直接贴结论，详情去看文章吧。

![k8s-network-feature-compare](http://7xrnwq.com1.z0.glb.clouddn.com/20171211-feature-comparation.jpg)


## 参考文章


[Comparison of Networking Solutions for Kubernetes](http://machinezone.github.io/research/networking-solutions-for-kubernetes/)

[Battlefield: Calico, Flannel, Weave and Docker Overlay Network](http://chunqi.li/2015/11/15/Battlefield-Calico-Flannel-Weave-and-Docker-Overlay-Network/)

[一个适合 Kubernetes 的最佳网络互联方案](http://dockone.io/article/1115)

[最新实践 | 将Docker网络方案进行到底](http://blog.shurenyun.com/shurenyun-docker-133/)

[Kubernetes网络原理及方案](http://www.youruncloud.com/blog/131.html)

[Docker容器跨主机通信方案选哪一种？](https://www.zhihu.com/question/49245479)


***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***