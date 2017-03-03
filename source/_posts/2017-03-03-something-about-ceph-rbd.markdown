---
layout: post
title:  "ceph-- 关于ceph rbd 的一些记录"
date:   2017-03-03 09:13:46
tags: ceph
---



## 引出

ceph 的中文文档做的真是非常棒，比起openstack的中文文档来说，不仅全，而且更新也比较及时，openstack的文档还是直接查阅英文版的比较好。

本文是在查阅ceph官网文档过程中，联系实际操作做出的一些思考与记录。


## ceph rbd 的简单说明

官网对rbd的定义是这样的：Ceph 块设备是精简配置的、大小可调且将数据条带化存储到集群内的多个 OSD。

ceph rbd是基于 rados来做的，也就是说，一个 rbd image 首先会被分成多个等大的 object，这些object再 根据 crush map 存储到对应的3个（或n个，n为奇数）OSD中，理解这一点对于理解 rbd 中的 stripe-unit和 stripe-count 有帮助。
所以如果考虑深层的话需要结合RADOS 的多种能力，如快照、复制和一致性。

对于openstack来说，通常是通过libvirt 来控制磁盘的虚拟化(比如qemu),再通过qemu来与rbd打交道，如下图：

![ceph-rbd-qemu](http://7xrnwq.com1.z0.glb.clouddn.com/2017-03-03-openstack-ceph-rbd.png)

qemu会调用librbd 实现rbd的增删改查等一系列操作，当然也有配套的 qemu命令，详见[QEMU 与块设备](http://docs.ceph.org.cn/rbd/qemu-rbd/)

libvirt 是一套虚拟机抽象层，它隐藏了底层多种虚拟化的具体实现，提供给client一套统一的通用API和shell接口。关于如何用 virsh 新建一个挂载rbd 的vm，可以参考[通过 LIBVIRT 使用 CEPH RBD](http://docs.ceph.org.cn/rbd/libvirt/)

关于openstack 与ceph 的整合与配置，参考[块设备与 OPENSTACK](http://docs.ceph.org.cn/rbd/rbd-openstack/)



## rbd 常用操作以及命令

rbd 的常用操作大致包括三个部分，一个是 常见的增删改查，一个是 针对 snapshot的操作，再加一个map/unmap操作。

### rbd的增删改查

```bash
$ rbd create --size {megabytes} {pool-name}/{image-name} # 创建 rbd

$ rbd ls {poolname} # 罗列 rbd

$ rbd info {pool-name}/{image-name} # 查看某一rbd具体信息

$ rbd resize --size 2048 foo (to increase) # 增加尺寸
$ rbd resize --size 2048 foo --allow-shrink (to decrease) # 减小尺寸

$ rbd rm {pool-name}/{image-name} # 删除rbd镜像
```



### rbd snapshot 相关操作

快照是映像在某个特定时间点的一份只读副本,ceph 的快照有一个比较高级的特性就是分层clone，在openstack 中会实现非常快的创建VM，下文会详细讲解。

#### rbd snapshot 的基本操作

```bash
$ rbd snap create {pool-name}/{image-name}@{snap-name} # 创建快照

$ rbd snap ls {pool-name}/{image-name} #罗列 某个镜像的快照

$ rbd snap rollback {pool-name}/{image-name}@{snap-name} # 快照回滚

$ rbd snap rm {pool-name}/{image-name}@{snap-name} # 删除某一快照

$ rbd snap purge {pool-name}/{image-name} # 清除某一镜像的所有快照

```

#### ceph 快照的分层 clone

Ceph 支持为某一rbd snapshot创建很多个写时复制（ COW ）克隆。所谓写时复制，就是执行 clone命令后，并没有立即执行clone操作，而是引用原文件，当发生写入操作时才真正执行clone操作。

![rbd-cow](http://7xrnwq.com1.z0.glb.clouddn.com/2017-03-03-ceph-cow.png)

注意： clone只能用于 protected 的snapshot,且仅支持克隆 format 2 的映像（即用 rbd create --image-format 2 创建的）。

分层快照使得 Ceph 块设备客户端可以很快地创建映像，这样便能实现openstack 中非常快的创建VM,而且，随着 openstack 的Nova image-create与ceph的snapshot的无缝连接，镜像的制作更快捷方便了，关于这部分，可以参考[基于Ceph RBD的OpenStack Nova快照](http://ceph.org.cn/2016/05/02/%E5%9F%BA%E4%BA%8Eceph-rbd%E7%9A%84openstack-nova%E5%BF%AB%E7%85%A7/)


clone 的 步骤如下：

![clone-process](http://7xrnwq.com1.z0.glb.clouddn.com/2017-03-03-clone-process.png)

相关命令如下：

```bash
$ rbd snap protect {pool-name}/{image-name}@{snapshot-name} # 保护快照

$ rbd clone {pool-name}/{parent-image}@{snap-name} {pool-name}/{child-image-name} #克隆快照

$ rbd snap unprotect {pool-name}/{image-name}@{snapshot-name} # 删除快照前，必须先取消保护。此外，不可以删除被克隆映像引用的快照，所以在删除快照前，必须先拍平（ flatten ）此快照的各个克隆。

$ rbd children {pool-name}/{image-name}@{snapshot-name} # 罗列快照的子孙

$ rbd flatten {pool-name}/{image-name} # 拍平克隆映像

```



### rbd map/unmap

```bash
$ rbd map {pool-name}/{image-name} --id {user-name} # 映射块设备

$ rbd showmapped # 查看已映射块设备

$ rbd unmap /dev/rbd/{poolname}/{imagename} # 取消块设备映射

```




## 参考文章


[CEPH 块设备](http://docs.ceph.org.cn/rbd/rbd/)


***END***
