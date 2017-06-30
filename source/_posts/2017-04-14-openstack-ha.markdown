---
layout: post
title:  "openstack-- 关于openstack HA 的一些记录"
date:   2017-04-14 15:10:32
tags: 
  - openstack
  - HA
---



## 概念

所谓HA(High Availability),高可用,就是指在本地系统的某个组件出现故障的情况下，系统依然可访问,应用的能力。
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

首先了解下rabbitmq集群的消息传递模式，rabbitmq的集群有两种模式，如下：

- 普通模式：默认的集群模式，以两个节点（rabbit01、rabbit02）为例来进行说明。对于Queue来说，消息实体只存在于其中一个节点rabbit01（或者rabbit02），rabbit01和rabbit02两个节点仅有相同的元数据，即队列的结构。当消息进入rabbit01节点的Queue后，consumer从rabbit02节点消费时，RabbitMQ会临时在rabbit01、rabbit02间进行消息传输，把A中的消息实体取出并经过B发送给consumer。所以consumer应尽量连接每一个节点，从中取消息。即对于同一个逻辑队列，要在多个节点建立物理Queue。否则无论consumer连rabbit01或rabbit02，出口总在rabbit01，会产生瓶颈。当rabbit01节点故障后，rabbit02节点无法取到rabbit01节点中还未消费的消息实体。如果做了消息持久化，那么得等rabbit01节点恢复，然后才可被消费；如果没有持久化的话，就会产生消息丢失的现象。该模式只是起到单纯的扩展作用（如增加queue的存储空间等），并没有达到HA 的效果。

- 镜像模式：将需要消费的队列变为镜像队列，存在于多个节点，这样就可以实现RabbitMQ的HA高可用性。作用就是消息实体会主动在镜像节点之间实现同步，而不是像普通模式那样，在consumer消费数据时临时读取。缺点就是，集群内部的同步通讯会占用大量的网络带宽。

rabbitmq的 A/P方案可以采用Pacemaker + （DRBD 或者其它可靠的共享 NAS/SNA 存储） + （CoroSync 或者 Heartbeat 或者 OpenAIS）来实现，安装配置可以参考[理解 OpenStack 高可用（HA）（5）：RabbitMQ HA](http://www.cnblogs.com/sammyliu/p/4730517.html).

openstack 官方推荐采用 rabbitmq 镜像模式实现A/A, 虽然是A/A，但是rabbitmq 镜像模式集群中还是有master/slave的概念,这个概念是针对queue的，通常创建queue的节点为该queue 的master节点，其他节点为slave节点。通常是一个master,多个slave ，master节点失效后，会自动选出另一个master。
对于消息的发布，producer可以连接任意节点，如果该节点不是master，则会转发给master，master会向其他slave节点发送该消息，后进行消息本地化处理，并组播复制消息到其他节点存储；对于consumer，可以选择任意一个节点进行连接，消费的请求会转发给master,为保证消息的可靠性，consumer需要进行ack确认，master收到ack后，才会删除消息，ack消息会同步(默认异步)到其他各个节点，进行slave节点删除消息。
可以看到虽然起到了H/A的效果，但是并没有达到减轻负载的作用，所以rabbitmq mirror 不支持负载均衡。
关于这部分详细讲解可以参考[Highly Available (Mirrored) Queues](https://www.rabbitmq.com/ha.html)


openstack rabbitmq H/A 的具体安装配置参考[Messaging service for high availability](https://docs.openstack.org/ha-guide/shared-messaging.html#install-rabbitmq)


### mysql 的HA

mysql 的HA 方案有很多，这里只讨论openstack 官方推荐的mariadb galara 集群。Galera Cluster 是一套在innodb存储引擎上面实现multi-master及数据实时同步的系统架构，业务层面无需做读写分离工作，数据库读写压力都能按照既定的规则分发到各个节点上去。在数据方面完全兼容 MariaDB 和 MySQL。 特点如下：

- 1）同步复制，（>=3）奇数个节点 
- 2）Active-active的多主拓扑结构
- 3）集群任意节点可以读和写
- 4）自动身份控制,失败节点自动脱离集群
- 5）自动节点接入
- 6）真正的基于”行”级别和ID检查的并行复制
- 7）无单点故障,易扩展

openstack doc 中提到在用HAproxy做galara cluster 的负载均衡时，因为该集群不支持跨节点对表加锁，也就是说如果OpenStack 某组件有两个会话分布在两个节点上同时写入某一条数据，会出现其中一个会话遇到死锁的情况。可以参考[Understanding reservations, concurrency, and locking in Nova](http://www.joinfu.com/2015/01/understanding-reservations-concurrency-locking-in-nova/),讲解的比较清晰。

![galera-lock](http://oeptotikb.bkt.clouddn.com/2017-04-15galera-update-certification-timeout.png)

解决方案上文作者就提出了一种方式，楼主看了下最新版代码已经没有with_lockmode('update') 语句。

安装配置参考[Database (Galera Cluster) for high availability](https://docs.openstack.org/ha-guide/shared-database.html)


## 网络控制节点 HA

这里说的网络控制节点是为了方便描述，并不是neutron的所有服务都在该节点，看下图，

![network-controller-node](http://oeptotikb.bkt.clouddn.com/2017-04-15-net-con.jpg)

可以认为neutron-service在控制节点，L3 agent,DHCP agent, Metadata agent,L2 agent等所在节点为网络控制节点，接下来就是针对具体的服务讨论HA 的方案，通常来说，我们只需考虑L3 agent,DHCP agent 这两个服务的HA即可，L2 agent只在所在的网络或者计算节点上提供服务，不需要HA。neutron-metadata-agent需要和 neutron-ns-metadata-proxy 通过soket 通信，可以在所有 neutron network 节点上都运行该 agent，只有 virtual router 所在的L3 Agent 上的 neutron-metadata-agent 才起作用，别的都standby。

### L3 agent HA

L3 agent 的HA ,官方给出的方案有两种，一种是利用VRRP协议（虚拟路由冗余协议）实现，另一种是DVR(分布式虚拟路由)实现。
关于VRRP协议，可以参考[Neutron L3 Agent HA 之 虚拟路由冗余协议（VRRP）](http://www.cnblogs.com/sammyliu/p/4692081.html), 具体配置以及网络的连接情况直接参考官方给的[guide](https://docs.openstack.org/newton/networking-guide/deploy-ovs-ha-vrrp.html)

同样，关于DVR的资料参考[Neutron 分布式虚拟路由（Neutron Distributed Virtual Routing）](http://www.cnblogs.com/sammyliu/p/4713562.html),以及官方[guide](https://docs.openstack.org/newton/networking-guide/config-dvr-ha-snat.html)

### DHCP agent HA

DHCP协议本身支持多个DHCP服务器，只需修改配置，为每个租户网络创建多个DHCP agent,即可实现HA。



## 存储控制节点 HA

cinder-volume的HA A/A方案目前还未实现，只能采取pacemaker 实现H/A，不过，随着openstack  Tooz 项目的开发完善，cinder-volume的A/A方案也渐渐明朗，就是采用openstack  Tooz  项目实现的分布式锁来实现。详细信息参考[cinder-volume如何实现AA高可用](http://int32bit.me/2017/03/16/cinder-volume%E5%A6%82%E4%BD%95%E5%AE%9E%E7%8E%B0AA%E9%AB%98%E5%8F%AF%E7%94%A8/)


## 计算节点 HA

包括计算节点和虚拟机的HA,社区从2016年9月开始一直致力于一个虚拟机HA的统一方案，
详细参考[igh Availability for Virtual Machines](http://specs.openstack.org/openstack/openstack-user-stories/user-stories/proposed/ha_vm.html).目前还是处于开发阶段。

业界目前使用的方案大致有以下几种：

- controller节点与compute节点通信（ping等），检查nova 服务运行状态，对于有问题的节点进行简单粗暴的evacuate.
- Pacemaker-remote： 突破Corosync的集群规模限制，参考[RDO的方案](https://github.com/beekhof/osp-ha-deploy/blob/master/ha-openstack.md)
- 集中式检查
- 分布式健康检查，参考[分布式健康检查：实现OpenStack计算节点高可用](http://www.infoq.com/cn/articles/OpenStack-awcloud-HA)



## 参考文章


[openstack doc](https://docs.openstack.org/ha-guide/intro-ha.html)

[世民谈云计算](http://www.cnblogs.com/sammyliu/p/4741967.html)

[Understanding reservations, concurrency, and locking in Nova](http://www.joinfu.com/2015/01/understanding-reservations-concurrency-locking-in-nova/)

[Fuel openstack HA guide](http://docs.ocselected.org/openstack-manuals/kilo/fuel-docs/contents.html#)

[RabbitMQ分布式集群架构和高可用性（HA）](http://blog.csdn.net/woogeyu/article/details/51119101)

[Highly Available (Mirrored) Queues](https://www.rabbitmq.com/ha.html)

***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***
