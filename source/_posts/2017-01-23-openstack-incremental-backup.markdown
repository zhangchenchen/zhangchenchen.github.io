---
layout: post
title:  "openstack系列--openstack 中的快照与备份"
date:   2017-01-23 14:54:25
tags: openstack
---

## 目录结构：

[引出](#A)

[nova 相关的快照/备份机制](#B)

[cinder相关的快照/备份机制 ](#C)



<a name="A"></a>

## 引出

- 备份作为一种常见的容灾机制，是系统设计中不可或缺的一环，openstack自然也不例外，本文列举了一些opensatck中常见的snapshot,backup等操作。
- 重点讨论nova,cinder。其他备份比如trove不再考虑
- 简单谈一下快照与备份的区别，快照保存的是瞬时数据，依赖于源数据，主要用于做重要升级之前先做个snapshot，以备出现差错回滚。备份主要是为了容灾，不依赖于源数据，且一般备份数据在不同数据中心。



<a name="B"></a>

## nova 相关的快照/备份机制

### Nova 快照
- 命令为nova image-create
- 严格来说，该快照并不是真正意义的快照，更像是创建一个镜像，所以命令为image-create.它并没有使用virt提供的virsh snapshot-create来创建snapshot，仅仅将虚拟机的系统磁盘文件完整的复制一份，然后把复制的文件上传至 glance 中作为镜像，之后二者毫无关联，类似于全量快照(没有考虑ceph作为统一存储)[check this](http://wsfdl.com/openstack/2014/08/12/Nova%E5%BF%AB%E7%85%A7%E5%88%86%E6%9E%90.html)。
- nova 中的快照支持冷快照和live snapshot,两者的命令都是 nova image-create。当满足QEMU 1.3+ and libvirt 1.0+版本要求时会使用live snapshot,否则使用冷快照。关于两者区别以及流程，[check this](http://wsfdl.com/openstack/2014/08/12/Nova%E5%BF%AB%E7%85%A7%E5%88%86%E6%9E%90.html)
- 如果是使用ceph作为统一存储(即nova,glance,cinder都将ceph rbd作为默认存储)，那么snapshot可以利用ceph的cow等特性实现更快速的快照，L版本已实现，流程大概如下：

![snapshot with rbd](http://oeptotikb.bkt.clouddn.com/2017-01-23-create_glance_snap.jpg)
这样做的坏处是会产生新建镜像对原虚拟机的依赖性，需要定期flatten。还不知道L版是怎么处理的，挖个坑，回头看下具体实现。详细参考[基于rbd提升虚机快照创建速度](http://niusmallnan.com/_build/html/_templates/openstack/rbd_snap_insteadof_qemu_snap.html)


### Nova 备份

- 命令为 nova backup <server> <name> <backup-type> <rotation>
- 其实该命令跟image-create的作用差不多，也是创建一个image,最大的区别就是它有一个rotation，会保证备份的个数为rotation个。详细的用法参考这个实例[Openstack - Nova Backup/Restore](http://qiita.com/idzzy/items/8b7833fc42b43a6db219)
- nova backup 不会自动每天/月周期执行，只能是我们自己命令，所以可以利用nova backup 命令写cron脚本或其他脚本实现自动化备份。这里有一个实例[OpenStack: Quick and automatic instance snapshot backup and restore (and before an apt upgrade) with nova backup](https://raymii.org/s/tutorials/OpenStack_Quick_and_automatic_instance_snapshot_backups.html)



<a name="C"></a>

## cinder相关的快照/备份机制

### cinder 快照 

- 命令为 cinder snapshot-create 同nova volume-snapshot-create一样效果。除了create,还有其他list,delete,metadata等命令。
- 可以用snapshot 对volume进行恢复，见示例[Openstack - Cinder Snapshot/Restore](http://qiita.com/idzzy/items/cfb568e83e2645e3f32e)  
- 如果cinder使用rbd作为后端存储，那么这时候的snapshot 便是利用rbd snapshot来实现的，关于ceph rbd snapshot的实现原理，参考[]()







### cinder 备份

- 命令 cinder backup-create 
- cinder支持多种backend,对全量备份，增量备份的支持也不尽相同，这里只讨论以ceph rbd 作为backend的情况，其他的backend可以参考这里，[Openstack 中cinder backup三种backend的对比](http://blog.csdn.net/wytdahu/article/details/45246095)- 使用 rbd作为后端，支持全量备份和增量备份。backup-create时先尝试创建增量备份，如果不成功，会创建全量备份，不需要指明专门的参数。
- 参考这篇, [cinder-backup 利用ceph实现增量备份](https://zhangchenchen.github.io/2017/05/09/openstack-cinder-incremental-backup-with-ceph/)。        







## 参考文章

[Nova 快照分析](http://wsfdl.com/openstack/2014/08/12/Nova%E5%BF%AB%E7%85%A7%E5%88%86%E6%9E%90.html)


[OpenStack: Quick and automatic instance snapshot backup and restore (and before an apt upgrade) with nova backup](https://raymii.org/s/tutorials/OpenStack_Quick_and_automatic_instance_snapshot_backups.html)

[基于rbd提升虚机快照创建速度](http://niusmallnan.com/_build/html/_templates/openstack/rbd_snap_insteadof_qemu_snap.html)

[OpenStack Nova snapshots on Ceph RBD](https://www.sebastien-han.fr/blog/2015/10/05/openstack-nova-snapshots-on-ceph-rbd/#disqus_comments)

[Openstack - Nova Backup/Restore](http://qiita.com/idzzy/items/8b7833fc42b43a6db219)

[openstack liberty版本使用 ceph 作为存储后端](https://www.zybuluo.com/zwei/note/352999)


[Openstack 中cinder backup三种backend的对比](http://blog.csdn.net/wytdahu/article/details/45246095)

[Inside Cinder’s Incremental Backup](http://gorka.eguileor.com/inside-cinders-incremental-backup/)

***END***
