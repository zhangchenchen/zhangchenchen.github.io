---
layout: post
title:  "ceph系列-- ceph 运维(一)-----常用命令"
date:   2016-12-20 11:06:25
tags: ceph
---

## 目录结构：

[ceph 数据组织方式](#A)

[ceph  增加/删除OSD](#B)

[ceph  增加/删除MON ](#C)

[操作 ceph集群中的服务 ](#D)

[监控 ceph 集群的一些命令](#E)

[ceph 用户管理](#F)

[ceph 数据管理命令](#G)



关于ceph 之前只是简单介绍过，并没有系统的去学习，总结过。之后几篇文章会详细介绍下ceph的常用运维命令。



<a name="A"></a>

## ceph 数据组织方式

 

首先看下ceph的数据组织方式，因为ceph与openstack结合时用到最多的便是块存储服务rbd，所以这里主要以rbd为例。
 
ceph 数据的组织方式是在物理资源的基础上进行了多层映射。

![ceph data organize](http://oeptotikb.bkt.clouddn.com/2016-12-20-ceph-data-organize.png)

客户想要创建一个rbd设备前，必须创建 一个pool，需要为这个pool指定pg的数量，在一个pool中的pg数量是不一定的，同时这个pool中要指明保存数据的副本数量3个副本。再在这个pool中创建一个rbd设备rbd0，那么这个rbd0都会保存三份，在创建rbd0时必须指定rbd的size，对于这个rbd0的任何操作不能超过这个size。之后会将这个块设备进行切块，每个块的大小默认为4M，并且每个块都有一个名字，名字就是object+序号。将每个object通过pg进行副本位置的分配，pg会寻找3个osd，把这个object分别保存在这三个osd上。osd上实际是把底层的disk进行了格式化操作，一般部署工具会将它格式化为xfs文件系统。最后对于object的存储就变成了存储一个文件rbd0.object1.file。

客户端使用rbd设备时，一般有两种方法。

 - 第一种 是kernel rbd。就是创建了rbd设备后，把rbd设备map到内核中，形成一个虚拟的块设备，这时这个块设备同其他通用块设备一样，一般的设备文件为/dev/rbd0，后续直接使用这个块设备文件就可以了，可以把/dev/rbd0格式化后mount到某个目录，也可以直接作为裸设备使用。这时对rbd设备的操作都通过kernel rbd操作方法进行的。

 
 - 第二种是librbd方式。就是创建了rbd设备后，这时可以使用librbd、librados库进行访问管理块设备。这种方式不会map到内核，直接调用librbd提供的接口，可以实现对rbd设备的访问和管理，但是不会在客户端产生块设备文件。

简单看下客户端写数据到osd的过程。

假设这次采用的是librbd的形式，使用librbd创建一个块设备，这时向这个块设备中写入数据，在客户端本地同过调用librados接口，然后经过pool，rbd，object、pg进行层层映射,在PG这一层中，可以知道数据保存在哪3个OSD上，这3个OSD分为主从的关系，也就是一个primary OSD，两个replica OSD。客户端与primay OSD建立SOCKET 通信，将要写入的数据传给primary OSD，由primary OSD再将数据发送给其他replica OSD数据节点。

![ceph-data-write](http://oeptotikb.bkt.clouddn.com/2016-12-20-ceph-data-write.png)

<a name="B"></a>

## ceph  增加/删除OSD

在官方文档里已经非常非常详细，[增加/删除 OSD](http://docs.ceph.org.cn/rados/operations/add-or-rm-osds/)

简单总结一下：

- 增加一个osd的流程大致：
    1. 预备工作：部署硬件，网络，安装ceph软件包
    2. 增加OSD:
        - 创建osd
        - 创建默认目录并挂载硬盘
        - 初始化 OSD 数据目录
        - 注册 OSD 认证密钥
        - 把 OSD 加入 CRUSH 图（crushmap其实就是部署物理环境的一个描述，比如哪几台机器在同一机架上，哪几个机架在同一机房里等等）
    3. 启动 OSD
        - 观察数据迁移

- 删除osd的流程：
    1. 踢出集群，置为out
    2. 观察数据迁移: ceph -w
    3. 停止 OSD 进程
    4. 删除 OSD：移出集群 CRUSH 图、删除认证密钥、删除 OSD 图条目、删除 ceph.conf 条目。如果主机有多个硬盘，每个硬盘对应的 OSD 都得重复此步骤。


<a name="C"></a>

## ceph  增加/删除MON

详见官方文档，[增加/删除监视器](http://docs.ceph.org.cn/rados/operations/add-or-rm-mons/)

- 增加一个mon流程：
    1. 预备工作：部署硬件，网络，安装ceph软件包
    2. 增加MON:
        - 在新监视器主机上创建mon默认目录
        - 获取监视器密钥环,保存到临时目录
        - 获取监视器运行图，保存到临时目录
        - 准备监视器的数据文件（包括密钥环，运行图文件）
        - 启动监视器，指定绑定端口
- 删除mon的流程：
    1. 停止监视器。
    2. 从集群删除监视器。
    3. 删除 ceph.conf 对应条目。





<a name="D"></a>

## 操作 ceph集群中的服务

ceph 中的服务包含 OSD ,MON两种，对这些服务的操作也无非是stop，start,restart等操作。有很多种方式，详见[操纵集群](http://docs.ceph.org.cn/rados/operations/operating/)

常用的是以service 的形式：

 - 启动所有守护进程： sudo service ceph -a start  (加 -a 即在所有节点上执行)
 - 启动一类守护进程： sudo service ceph start osd (只在本节点启动OSD进程)
 - 启动单个守护进程： sudo service ceph start osd.0



<a name="E"></a>

## 监控 ceph 集群的一些命令


- 检查鸡群状态：ceph status 或 ceph -s
- 实时检查集群：ceph -w
- 检查集群的容量使用情况：ceph df
- 检查OSD状态：ceph osd stat 或 ceph osd tree
- 检查MON状态： ceph mon stat
- 查看MON 选举状态：ceph quorum_status
- 检查归置组（PG）状态 ：ceph pg stat
    1. 归置组的状态有多个，详见[监控 OSD 和归置组](http://docs.ceph.org.cn/rados/operations/monitoring-osd-pg/)
- 管理套接字命令允许你在运行时查看和修改配置

<a name="F"></a>

## ceph 用户管理

- 罗列用户：ceph auth list

- 用户的增删修改参见 [用户管理](http://docs.ceph.org.cn/rados/operations/user-management/)


<a name="G"></a>

## ceph 数据管理命令

只说下最常见的三种数据：pool,pg,和crushmap 的管理。

- pool 存储池管理：Ceph 在存储池内存储数据，它是对象存储的逻辑组；存储池管理着归置组数量、副本数量、和存储池规则集。要往存储池里存数据，用户必须通过认证、且权限合适，存储池可做快照。主要操作如下：
    1. 列出存储池： ceph osd lspools
    2. 创建存储池： ceph osd pool create {pool-name} {pg-num} [{pgp-num}] [replicated] [crush-ruleset-name] [expected-num-objects]
    3. 设置存储池配额： ceph osd pool set-quota {pool-name} [max_objects {obj-count}] [max_bytes {bytes}]
    4. 删除存储池： ceph osd pool delete {pool-name} [{pool-name} --yes-i-really-really-mean-it]
    5. 重命名存储池： ceph osd pool rename {current-pool-name} {new-pool-name}
    6. 查看存储池统计信息： rados df
    7. 拍下存储池快照： ceph osd pool mksnap {pool-name} {snap-name}
    8. 删除存储池快照：ceph osd pool rmsnap {pool-name} {snap-name}
    9. 设置对象副本数：ceph osd pool set {poolname} size {num-replicas}
    10. 获取对象副本数：ceph osd dump | grep 'replicated size'

- 归置组(Placement Group)管理：Ceph 把对象映射到归置组（ PG ），归置组是一逻辑对象池的片段，这些对象组团后再存入到 OSD 。归置组减少了各对象存入对应 OSD 时的元数据数量，根据N副本策略，每个归置组对应N个OSD。主要的操作如下：

    1. 设置归置组数量：ceph osd pool set {pool-name} pg_num {pg_num}
    虽然 pg_num 的增加引起了归置组的分割，但是只有当用于归置的归置组（即 pgp_num ）增加以后，数据才会被迁移到新归置组里，所以还要增加一步：
    ceph osd pool set {pool-name} pgp_num {pgp_num}
    2. 获取归置组数量：ceph osd pool get {pool-name} pg_num
    3. 获取归置组统计信息：ceph pg dump [--format {format}]
    4. 获取一归置组运行图：ceph pg map {pg-id}
    5. 获取一 PG 的统计信息：ceph pg {pg-id} query
- CRUSH 图(CRUSH Map):  CRUSH 是重要组件，它使 Ceph 能伸缩自如而没有性能瓶颈、没有扩展限制、没有单点故障，它为 CRUSH 算法提供集群的物理拓扑，以此确定一个对象的数据及它的副本应该在哪里、怎样跨故障域存储，以提升数据安全。详见[CRUSH 图](http://docs.ceph.org.cn/rados/operations/crush-map)。 
编辑crushmap的步骤如下：
    1. 获取 CRUSH 图：ceph osd getcrushmap -o {compiled-crushmap-filename}
    2. 反编译 CRUSH 图：crushtool -d {compiled-crushmap-filename} -o {decompiled-crushmap-filename}
    3. 至少编辑一个设备、桶、规则
    4. 编译 CRUSH 图：crushtool -c {decompiled-crush-map-filename} -o {compiled-crush-map-filename}
    5. 注入 CRUSH 图：ceph osd setcrushmap -i  {compiled-crushmap-filename}


## 参考文章

[ceph的数据存储之路(2) ----- rbd到osd的数据映射](https://my.oschina.net/u/2460844/blog/531686)

[集群运维--ceph doc](http://docs.ceph.org.cn/rados/operations/)


***END***
