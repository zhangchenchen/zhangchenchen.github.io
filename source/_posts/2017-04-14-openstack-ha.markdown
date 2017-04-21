---
layout: post
title:  "openstack-- 关于openstack HA 的一些记录"
date:   2017-04-14 15:10:32
tags: 
  - openstack
  - HA
---



## 概念

所谓HA(High Availability),高可用，就是指在本地系统的某个组件出现故障的情况下，系统依然可访问,应用的能力。
而为了达到这种目的，我们一般采用节点冗余的方法组成集群来提供服务。通常来说，可以分为两种冗余的方式：
- Active/Passive HA: 即A/P模式，中文主备模式，通常会有一个主节点来提供服务，备用节点会同步/异步与主节点进行数据同步。当主节点故障时，备用节点会启用代替主节点。通常会使用CRM软件比如Pacemaker来进行主备节点之间的切换并提供一个VIP提供服务。
- Active/Active HA:也就是我们常说的A/A方式，中文也称作双活或多主模式。这种模式下，系统在集群的所有服务器上运行同样的负载，也就是说集群内服务器的功能都是一样的。这种模式下，通常会采用负载均衡软件如HAProxy提供VIP进行负载均衡。

涉及到具体的服务时，HA的部署方式也不尽相同，一般来说，一些无状态的服务比如提供http请求转发的http服务器等，可以直接部署A/A模式，无需考虑数据不一致的问题。而一些有状态的服务，比如数据库等，则需要具体问题具体对待，如果这类服务提供原生的A/A服务，那么尽量利用他们提供的原生A/A服务。如果实在不行那么只能从A/P角度去部署。



## 关于openstack 的HA

根据openstack部署节点的功能来分，可以将各节点分为以下四种，这四种节点的HA方式也不尽相同：
- Cloud Controller Node （云控制节点）：安装各种 API 服务和内部工作组件（worker process）。同时，通常也会将共享的 DB 和 MQ 安装在该节点上。
- Neutron Controller Node （网络控制节点）：安装 Neutron L3 Agent，L2 Agent，LBaas，VPNaas，FWaas，Metadata Agent 等 Neutron 组件。
- Storage Controller Node （存储控制节点）：安装 Cinder volume 以及 Swift 组件。
- Compute node （计算节点）：安装 Nova-compute 和 Neutron L2 Agent，在该节点上创建虚机。


本文接下来就主要按照这四个节点的顺序记录下对应的HA方案，通常来说，部署的原则遵循尽量A/A,尽量采用原生支持的HA方案,考虑负载均衡，以及方案尽量简单。



## 云控制节点 HA

注：这里只考虑A/A方案，A/P方案可以通过Pacemaker+Corosync 的方式搭建集群，业界应用的比较少。

首先看下控制节点中的各种服务，对于无状态服务（API服务+内部工作组件如nova-scheduler等），我们可以直接多节点部署，如果是提供API 的利用pacemaker 提供VIP,HAproxy提供负载均衡。对于有状态的服务我们可以尽量使用原生的AA方案，罗列如下（摘自[理解 OpenStack 高可用（HA）（1）：OpenStack 高可用和灾备方案 [OpenStack HA and DR]](http://www.cnblogs.com/sammyliu/p/4741967.html)）：


- API 服务：包括 *-api, neutron-server，glance-registry(注：glance api v2 版本将该服务与glance-api集合到了一起), nova-novncproxy，keystone，httpd 等。由 HAProxy 提供负载均衡，将请求按照一定的算法转到某个节点上的 API 服务。由  Pacemaker 提供 VIP。
- 内部组件：包括 *-scheduler，nova-conductor，nova-cert 等。它们都是无状态的，因此可以在多个节点上部署，它们会使用 HA 的 MQ 和 DB。
RabbitMQ：跨三个节点部署 RabbitMQ 集群和镜像消息队列。可以使用 HAProxy 提供负载均衡，或者将 RabbitMQ host list 配置给 OpenStack 组件（使用 rabbit_hosts 和 rabbit_ha_queues 配置项）。
- MariaDB：至少跨三个节点部署 Gelera MariaDB 多主复制集群。由 HAProxy 提供负载均衡。
- HAProxy：向 API，RabbitMQ 和 MariaDB 多活服务提供负载均衡，其自身由 Pacemaker 实现 A/P HA，提供 VIP，某一时刻只由一个HAProxy提供服务。在部署中，也可以部署单独的 HAProxy 集群。
- Memcached：它原生支持 A/A，只需要在 OpenStack 中配置它所有节点的名称即可，比如，memcached_servers = controller1:11211,controller2:11211。当 controller1:11211 失效时，OpenStack 组件会自动使用controller2:11211。

![openstack-con-ha](http://oeptotikb.bkt.clouddn.com/2017-04-21-openstack-con-ha.jpg)

从一个请求的角度大致时序如下：

![flow](http://oeptotikb.bkt.clouddn.com/2017-04-21-flow.jpg)

这里在重点说一下rabbitmq 与mysql用的HA方案：

### rabbitmq 的HA






### mysql 的HA







## 网络控制节点 HA



## 存储控制节点 HA





## 计算节点 HA






## 参考文章


[openstack doc](https://docs.openstack.org/ha-guide/intro-ha.html)

[世民谈云计算](http://www.cnblogs.com/sammyliu/p/4741967.html)

[Understanding reservations, concurrency, and locking in Nova](http://www.joinfu.com/2015/01/understanding-reservations-concurrency-locking-in-nova/)

[Fuel openstack HA guide](http://docs.ocselected.org/openstack-manuals/kilo/fuel-docs/contents.html#)


***END***
