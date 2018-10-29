---
layout: post
title:  "openstack-- cinder-backup 利用ceph实现增量备份"
date:   2017-05-09 14:21:32
tags: 
  - openstack
  - ceph
  - incremental backup
---




## 配置

首先如果想要实现 ceph backend cinder的增量备份的话，要保证cinder-volume 与 cinder-backup 的backend 都是ceph，涉及到具体实践的话就是openstack 与 ceph  integration 的 配置，ceph官网有详实的步骤 [BLOCK DEVICES AND OPENSTACK](http://docs.ceph.com/docs/master/rbd/rbd-openstack/) ,当然，中文版的一搜也一大把[openstack的glance、nova、cinder使用ceph做后端存储](http://www.cnblogs.com/pycode/p/6494885.html) ,这里提一点，就是 ，如果cinder-volume 是默认以ceph rbd为backend，那么glance 配置文件其实也可以直接默认用cinder 作为默认存储，但最终其实都是存在ceph 里，glance配置文件可以类似这样，这样cinder list 也可以列出镜像盘：

```bash
   ......
[glance_store]
stores=file,http,cinder,rbd
default_store=cinder
cinder_os_region_name=RegionOne
.....

```

还想提一点就是，backup 尽量使用 multi-ceph 的情况，因为个人感觉备份主要是为了容灾，单 ceph的情况其实容灾能力有限。


## 使用


注：以Liberty版本示例
使用非常简单，首先对 volume 做一个全量备份：

```bash
$ cinder backup-create  --name fullbackup <volume-ID>
```

然后再做一次备份就是增量备份了，

```bash
$ cinder backup-create  --name incrementalbackup --incremental <volume-ID>
```

其实不用加incremental tag也是可以的。

可以利用删除backup 来证明一下，增量备份都会有一个 parent backup（父备份），就是基于父备份来做增量备份，通常来说，就是我们刚开始做的那个全量备份。在删除父备份之前，必须先把所有它的子备份删除，才可以成功删除父备份。示例如下：

![cinder-backup](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/2017-05-09-cinder-backup.png)



## 实现原理 

关于备份的具体流程可以参考这张图：

![backup-workflow](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/backup_workflow.png)

![detail-backup](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/2017-05-10-backup-detail.jpg)

熟悉ceph的话，应该知道ceph rbd的export-diff，import-diff 功能：

- export-diff ：将某个 rbd image 在两个不同时刻的数据状态比较不同后导出补丁文件。
- import-diff :将某个补丁文件合并到某个 rbd image 中。

ceph 增量备份就是基于这两个基本功能，详细命令示例可以参考[rbd的增量备份和恢复](http://www.zphj1987.com/2016/06/22/rbd%E7%9A%84%E5%A2%9E%E9%87%8F%E5%A4%87%E4%BB%BD%E5%92%8C%E6%81%A2%E5%A4%8D/)


### 首次备份

- 在用作备份的ceph集群中新建一个base image，与源 volume 大小一致，name 的形式为 "volume-VOLUMD_UUID.backup.base"
- 源 rbd image 新建一个快照，name形式为 backup.BACKUP_ID.snap.TIMESTRAMP
- 源rbd image 上使用 export-diff 命令导出从刚开始创建时到上一步快照时的差异数据，其实就是现在整个rbd的数据，然后通过管道将差量数据导入刚刚在备份集群上新创建的base image中

### 再次备份

- 在要备份的源volume 中找到满足 r"^backup\.([a-z0-9\-]+?)\.snap\.(.+)$" 的最近一次快照。
- 源volume 创建一个新的快照，name 形式为 backup.BACKUP_ID.snap.TIMESTRAMP。
- 源rbd image 上使用 export-diff 命令导出与最近的一次快照比较的差量数据，然后通过管道将差量数据导入到备份集群的rbd image中

恢复时相反，只需要从备份集群找出对应的快照并导出差量数据，导入到原volume即可




## 参考文章

[BLOCK DEVICES AND OPENSTACK](http://docs.ceph.com/docs/master/rbd/rbd-openstack/)

[openstack的glance、nova、cinder使用ceph做后端存储](http://www.cnblogs.com/pycode/p/6494885.html)

[Ceph backup driver](https://docs.openstack.org/ocata/config-reference/block-storage/backup/ceph-backup-driver.html)

[Cinder磁盘备份原理与实践](http://int32bit.me/2017/03/30/cinder%E7%A3%81%E7%9B%98%E5%A4%87%E4%BB%BD%E5%8E%9F%E7%90%86%E5%92%8C%E5%AE%9E%E8%B7%B5/)

[ Openstack 中cinder backup三种backend的对比](http://blog.csdn.net/wytdahu/article/details/45246095)

[Inside Cinder’s Incremental Backup](https://gorka.eguileor.com/inside-cinders-incremental-backup/)

***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***
