---
layout: post
title:  "Kuberbetes-- 私有容器云平台实践优化总结"
date:   2017-12-08 11:03:10
tags: 
  - Kubernetes
  - private cloud
---


## 概述

私有容器云平台一期的项目算是进入尾声，本篇文章算是对该项目做一个简单的总结，以及对基于k8s私有容器云落地的一些想法。


## 架构

整体大致架构如下：
![cloud-arch](http://oeptotikb.bkt.clouddn.com/k8s-arch.jpg)

因为没有公有云的加入，算是一个比较简单的架构，基本功能包括容器应用的构建，管理，镜像管理（采用Harbor），权限管理，以及针对集群的监控，日志管理等等。因为是第一版，没有将devops放进来，可能年前会发一个小版本加进来。

主要的开发工作如下：
- 集群的安装部署，以及对应自动化脚本文件的编写（我们用的是saltstalk），包括k8s，ceph，harbor等等。
- 定制k8s addon文件，包括calico网络，监控，日志等。
- 定制k8s yaml模板。
- 用户 web端的编码，用户对容器云的使用都是通过该web client进行，包括权限管理也是放在这里，可以通过web shell访问容器。
- 应用迁移
- 其他

## 优化

结合具体工作谈一下具体优化的地方：

### docker部分

- 首先是版本问题，k8s建议是 1.12.6,同时要注意与内核版本的兼容问题。
- docker 镜像部分优化：docker镜像大致可以分为三层，底层的系统层，中间层是运行环境（java/python等），最上层是应用代码层。一般来说系统层肯定是会复用的，应用层肯定是不会复用的。所以系统层就需要做一下定制，安装一些常用工具即可。中间层的复用就见仁见智了，如果想节省磁盘空间，且应用的运行环境比较统一（比如都是java8），那么可以复用（将jdk直接以volume形式挂载到容器中），如果业务繁多，且技术债务比较重，还是隔离较好。（以笔者的经验，若不是推倒重来，一般还是一个容器一个环境更好。）
- 依赖库的安装：尽量使用yum等安装（装完后会默认删除）

### kubernetes 部分

#### 安装部署

这部分没什么好说的，就是尽量上tls（可以使用cloudera的套件），然后master节点做HA，我们使用kubeadm进行的安装部署，然后在此基础上做的HA 。
最近好像使用rancher 安装更简单了，笔者抽时间会尝试一下。

#### 使用

- 网络部分：k8s网络方案有很多，根据适用场景的不同，选择也不同。我们用的calico，因为对openstack比较友好。目前来看，比较有潜力的开源方案有flannel(host-gw),calico(ipip/bgp)。关于网络这一部分，后续会专门写一篇文章。
- 存储部分：这部分我们使用了多种存储以应对不同的应用类型。对于一般的有状态服务，我们使用ceph rbd。如果是对io请求敏感的服务，我们会挂载本地host ssd。这个地方需要注意做node label，防止应用挂掉然后schedule到别的host。
- QOS部分：QOS部分就使用k8s的resource request/limit来控制。
- label使用：随着应用的逐渐增多，各种label层出不穷，而且运维人员不可能对每一个label都熟悉，所以建议对label做一个管理或规范。

#### addon插件

- 监控：Prometheus & Grafana，基本够用，参考[kubernetes-- kubernetes 监控指南（二）](https://zhangchenchen.github.io/2017/11/09/kubernetes-monitoring-guide-2/)
- 日志：EFK基本够用，如果不放心，中间加一个MQ（kafka）。对于日志收集，我们是在在每个pod中添加一个sidecar container将输出日志都重定向到标准输出，这样就可以用fluentd统一收集。
- web ui：除了原生的dashboard（只允许特定的几个运维IP访问）外，我们还做了一个面向developer的界面，用于执行应用的创建，升级等操作。
- devops：这部分还没做，年前会出个小版本。

## 后续的工作以及一些想法

- devops部分，目前我们只能算是拿容器当虚拟机来使用。只有上了devops，才算真正发挥容器的作用。后续会对接gitlab代码库，调研devops工具，选择合适的方案。
- 目前我们云平台的权限管理部分还太粗粒度，只是在web 层面做的一刀切，后续会对接k8s 的RBAC。
- 之后会对接大数据的一部分业务，而且目前社区spark on kubernetes也在积极开发，后续会持续关注。
- 支持更多一键式部署应用。
- 对k8s的一些概念要进行二层封装，从使用者的角度考虑，做到看一眼界面基本知道大致怎样用。


***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***