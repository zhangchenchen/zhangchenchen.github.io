---
layout: post
title:  "Kuberbetes-- kubernetes网络方案总结（二）"
date:   2017-12-14 14:00:11
tags: 
  - Kubernetes
---


## 概述

上篇文章对k8s的几种网络方案做了详细对比，本篇文章主要探究下两种典型的网络方案flannel和calico。

## flannel

flannel是coreos为k8s设计的一个非常简洁的多节点3层容器网络互联方案。

### 原理

flannel 旨在解决不同host上的容器网络互联问题，大致原理是每个 host 分配一个 subnet，容器从此 subnet 中分配 IP，这些 IP 可以在 host 间路由，容器间无需 NAT 和 port mapping 就可以跨主机通信。每个 subnet 都是从一个更大的 IP 池中划分的，flannel 会在每个主机上运行一个叫 flanneld 的 agent，其职责就是从池子中分配 subnet。为了在各个主机间共享信息，flannel 用 etcd（如果是k8s集群会直接调用k8s api）存放网络配置、已分配的 subnet、host 的 IP 等信息。节点间的通信有以下多种backen支持。
- VXLAN：推荐配置，利用内核级别的VXLAN来封装host之间传送的包。
- host-gw：对于性能有要求的推荐配置，但是不支持云环境。通过在host的路由表中直接创建到其他主机 subnet 的路由条目，从而实现容器跨主机通信。要求所有host在二层互联。
- udp：默认模式，通常用于debug，或以上两种条件都不具备。

找了一张有容云博客中的图：

![flannel](http://oeptotikb.bkt.clouddn.com/20171212140335-flannel.jpg)

图中backend是用udp作为示例，我们可以换成其他任意两种，原理类似。


### 实践

在docker 中的实践可以参考[安装配置 flannel - 每天5分钟玩转 Docker 容器技术（59）](http://www.cnblogs.com/CloudMan6/p/7424858.html)系列文章。

在k8s中，如果是使用kubeadm安装k8s集群的话，可以参考[kubeadm](https://github.com/coreos/flannel/blob/master/Documentation/kubernetes.md),其实最主要的就是[这个文件](https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml)，默认是vxlan，博主之前安装就是用的vxlan,就不重复操作了，大致解释下yaml文件中的内容：
- 创建对应clusterrole,clusterrolebinding。
- 创建一个flannel service account。
- 创建一个configmap用作cni和flannel的配置。其中flannel的network 要与之前kubeadm安装k8s时的pod network CIDR符合，默认backend是vxlan，可以改成其他的。
- 创建flannel daemonset在每台host上起一个pod，每个pod有两个容器，一个跑flanneld 程序，另一个initcontainer部署一些cni配置以便kubelet可以读取。

## calico 

相对于flannel的小而美，calico是一个比较完整的项目，官网的title是“Secure networking for the cloud native era”，可见calico是专为云环境设置，且比较注重安全性，也就是网络隔离，ACL控制等都是可以实现的。列一下官网中的feature:

- 可扩展，分布式的控制组件
- 基于可配置policy的网络安全
- 无需overlay
- 可以集成所有的云平台（From Kubernetes to OpenStack, AWS to GCE）
- 经过生产环境认证。

### 原理

calico是一个纯三层的数据中心网络方案，实现类似于flannel host-gw,不过它没有复用docker 的docker0 bridge，而是自己实现的。
Calico在每一个计算节点利用Linux Kernel实现了一个高效的vRouter来负责数据转发，而每个vRouter通过BGP协议负责把自己上运行的workload的路由信息像整个Calico网络内传播——小规模部署可以直接互联，大规模下可通过指定的BGP route reflector来完成。
Calico基于iptables还提供了丰富而灵活的网络Policy，保证通过各个节点上的ACLs来提供Workload的多租户隔离、安全组以及其他可达性限制等功能。
对于有IP限制的host，也可以使用calico的IPIP方案（overlay方式）。

calico 的架构：
![calico-arch](http://oeptotikb.bkt.clouddn.com/20171212-calico-arch.jpg)

具体通信流程可以结合实践，参考[如何部署 Calico 网络？- 每天5分钟玩转 Docker 容器技术（67）](http://www.cnblogs.com/CloudMan6/p/7509975.html)


### 实践

在docker上的实践参考[Calico with Docker](https://docs.projectcalico.org/v2.6/getting-started/docker/),嫌麻烦直接看[如何部署 Calico 网络？- 每天5分钟玩转 Docker 容器技术（67）](http://www.cnblogs.com/CloudMan6/p/7509975.html)

在k8s上的部署可以参考[Adding Calico to an Existing Kubernetes Cluster](https://docs.projectcalico.org/v1.5/getting-started/kubernetes/installation/#kubernetes-hosted-installation)

tips: calico 可以通过profile 来实现ACL控制，结合k8s的NetworkPolicy可以实现租户网络隔离,这对公有云还是很有必要的，参考[容器编排之Kubernetes多租户网络隔离](https://zhuanlan.zhihu.com/p/26614324)。

## 参考文章


[coreos/flannel](https://github.com/coreos/flannel/blob/master/Documentation/kubernetes.md)

[calico](https://www.projectcalico.org/#getstarted)

[Kubernetes网络原理及方案](http://www.youruncloud.com/blog/131.html)

[安装配置 flannel - 每天5分钟玩转 Docker 容器技术（59）](http://www.cnblogs.com/CloudMan6/p/7424858.html)

[最新实践 | 将Docker网络方案进行到底](http://blog.shurenyun.com/shurenyun-docker-133/)



***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***