---
layout: post
title:  "Kuberbetes-- 部署生产级MySQL到Kubernetes集群中"
date:   2018-05-10 11:02:11
tags: 
  - Kubernetes
  - Mysql
---


## 概述

K8s天然支持无状态应用的自动化运维（如HA,Scale等），对于有状态应用就相对比较麻烦了，本文以Mysql为例，详细梳理一下有状态应用在K8S中的部署运维。

## Mysql HA 方案

有状态应用在K8S中难以运维的原因是该应用本身就比较难运维（不然就不会有DBA 这样一个专门的职位了），当涉及到有状态应用时，你要考虑数据的存储，可用性，扩展性，事务，灾备等等。不过要想在K8S中部署生产级别的有状态应用，首先要知道在没有使用K8S时生产级别的应用架构方案。
这里还是以Mysql为例，简单说下常用的几种生产环境中的Mysql 架构方案。
本人不是专业的DBA,而且本身Mysql HA就是水很深的一个方向，理解有限，这里只谈大致方案，不说技术细节。
Mysql HA架构有很多种，具体选型时要考虑架构的HA level，支撑应用的性质，具体的部署环境等。 这里根据Mysql DOC中的分类来具体说明（个人感觉这个分类也不是太精确），官网根据能达到的HA level分了三个层次，分别是：
- Data Replication.
- Clustered & Virtualized Systems.
- Shared-Nothing, Geographically-Replicated Clusters.

这三个层次的可用性依次递增，不过复杂性也随之增加。
![tradeoff](http://7xrnwq.com1.z0.glb.clouddn.com/20180510142133tradeoff.jpg)

### Data Replication

这个级别的架构主要是 replication的方法实现mysql数据的复制冗余，具体方案有如下几种：

#### MYSQL 主从或主主半同步复制

![replication](http://7xrnwq.com1.z0.glb.clouddn.com/20180510150744-replication.jpg)

这种架构比较简单，使用原生半同步复制作为数据同步的依据,缺点是需要额外考虑haproxy、keepalived的高可用机制，而且完全依赖于半同步复制，如果半同步复制退化为异步复制，数据一致性无法得到保证，可以通过针对网络波动的半同步复制优化解决。


#### 利用MHA 实现HA

MHA（Master High Availability）目前在MySQL高可用方面是一个相对成熟的解决方案，它由日本DeNA公司youshimaton（现就职于Facebook公司）开发，是一套优秀的作为MySQL高可用性环境下故障切换和主从提升的高可用软件。它主要解决的是master fail-over的情况下实现秒级切换且保证数据一致性。

![mha](http://7xrnwq.com1.z0.glb.clouddn.com/20180510151622-mha1.jpg)

优点是操作非常简单，可以将任意slave提升为master，且MHA可以设定多个master,可用性提高，缺点是逻辑较为复杂，发生故障后排查问题，定位问题更加困难，且用perl开发，二次开发困难。
除了MHA ,还有MMM(Master-Master replication managerfor Mysql，Mysql主主复制管理器)方案。

#### SAN共享存储实现HA

![san](http://7xrnwq.com1.z0.glb.clouddn.com/20180510152934-san.jpg)

这种方案因为使用共享存储，不需要数据同步，不过专门的共享存储花销比较大，且需要专门的运维人员，适合一些土豪公司。

#### DRBD磁盘复制实现HA

DRBD是一种基于软件、基于网络的块复制存储解决方案，主要用于对服务器之间的磁盘、分区、逻辑卷等进行数据镜像，当用户将数据写入本地磁盘时，还会将数据发送到网络中另一台主机的磁盘上，这样的本地主机(主节点)与远程主机(备节点)的数据就可以保证实时同步。
穷人版的SAN ,唯一不同的是没有使用SAN网络存储 ，而是使用local disk。由于是实时复制磁盘数据，性能会有影响。

![drbd](http://7xrnwq.com1.z0.glb.clouddn.com/20180510153049-drbd.jpg)


### Clustered & Virtualized Systems

大致有两种方案，一种是基于Mysql的NDB CLUSTER，另一种是MarianDB的Galera方案。

#### Mysql NDB CLUSTER

![ndb](http://7xrnwq.com1.z0.glb.clouddn.com/20180510160922-ndb.jpg)

国内用NDB集群的公司非常少。NDB集群不需要依赖第三方组件，全部都使用官方组件，能保证数据的一致性，某个数据节点挂掉，其他数据节点依然可以提供服务，管理节点需要做冗余以防挂掉。
缺点是：管理和配置都很复杂，而且某些SQL语句例如join语句需要避免。

#### MarianDB Galera Cluster

![galera](http://7xrnwq.com1.z0.glb.clouddn.com/20180510161231-galera.jpg)

MariaDB Galera Cluster 是一套在mysql innodb存储引擎上面实现multi-master及数据实时同步的系统架构，业务层面无需做读写分离工作，数据库读写压力都能按照既定的规则分发到 各个节点上去。在数据方面完全兼容 MariaDB 和 MySQL。 

### Geographically-Replicated Clusters

这种级别的Mysql集群是跨地域的数据中心，其实是上述解决方案的一种scale，但是规模更大，架构更加复杂，也没有一种统一的解决方案，这里不做讨论。

## 部署Mysql 到K8S集群 




## 参考文章

[Best practices for MySQL High Availability ](https://www.slideshare.net/bytebot/best-practices-for-mysql-high-availability)

[Choosing the right MySQL High Availability Solution](http://www.clusterdb.com/mysql/choosing-the-right-mysql-high-availability-solution-webinar-replay)

[High Availability and Scalability](https://dev.mysql.com/doc/mysql-ha-scalability/en/ha-overview.html)

[五大常见的MySQL高可用方案](https://zhuanlan.zhihu.com/p/25960208)

[mysql复制高可用方案总结](http://www.fblinux.com/?p=1044)

[MySQL高可用各个技术的比较](http://database.51cto.com/art/201504/473788.htm)

[用 Go 搭建 Kubernetes Operators](https://juejin.im/entry/59edb5656fb9a0452404fd78)

[mysql on kubernetes](https://schd.ws/hosted_files/kccncna17/cc/MySQL%20on%20Kubernetes.pdf)

[容器化RDS：计算存储分离还是本地存储？](http://dockone.io/article/3673)

***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***