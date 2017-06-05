---
layout: post
title:  "ceph-- ceph rbd快照的实现原理"
date:   2017-06-05 15:21:32
tags: 
  - ceph
  - snapshot
---



## 引出

关于openstack 中的快照，备份部分之前做过一次总结，[openstack 中的快照与备份](https://zhangchenchen.github.io/2017/01/23/openstack-incremental-backup/) ,因为目前基本使用ceph rbd作为共享存储后端，所以本篇文章想讨论以rbd作为存储后端时的备份与快照情况：


- nova image-create：具体实现参考[openstack 中的快照与备份](https://zhangchenchen.github.io/2017/01/23/openstack-incremental-backup/) 
- nova backup: 同上。
- cinder snapshot：直接利用 rbd snapshot 实现
- cinder backup：支持增量备份，也用到了rbd snapshot，参考[ cinder-backup 利用ceph实现增量备份](https://zhangchenchen.github.io/2017/05/09/openstack-cinder-incremental-backup-with-ceph/)


由以上情况可知，rbd snapshot 几乎都要用到，所以接下来探讨下rbd snapshot 的实现原理。

## rbd snapshot vs rbd clone

首先知道这两者的区别：

- snapshot：是某一镜像(image) 在特定时间点的一个只读副本,注意，是只读副本，所以该snapshot是无法作为一个image去进行写操作的（比如起一个虚拟机）。
- clone: 是针对snapshot的一种操作，可以理解为对snapshot的再度拷贝，从而实现可写操作，简单讲就是将image的某一个snapshot 的状态复制变成另一个image，该image状态与snapshot完全一致，但是可读可写（可以从此image起一个虚拟机）。

弄懂了两者的区别，接下来需要认真探讨下两者实现的原理：

### rbd snapshot 实现原理

注意：snapshot时，一定要对原rbd image 停止IO操作，否则会引起数据不一致。

![snapshot](http://7xrnwq.com1.z0.glb.clouddn.com/snapshot.png)

跟git 实现版本控制类似，使用 COW （copy on write）方式实现。这里叙述下对某一pool中的某个image 进行snapshot 的过程：

直接引用自[解析Ceph: Snapshot](http://www.wzxue.com/%E8%A7%A3%E6%9E%90ceph-snapshot/)：

>>>每个Pool都有一个snap_seq字段，该字段可以认为是整个Pool的Global Version。所有存储在Ceph的Object也都带有snap_seq，而每个Object会有一个Head版本的，也可能会存在一组Snapshot objects，不管是Head版本还是snapshot object都会带有snap_seq，那么接下来我们看librbd是如何利用该字段创建Snapshot的。
 1. 用户申请为”pool”中的”image-name”创建一个名为”snap-name”的Snapshot
 2. librbd向Ceph Monitor申请得到一个”pool”的snap sequence，Ceph Monitor会递增该Pool的snap_seq，然后返回该值给librbd。
 3. librbd将新的snap_seq替换原来image的snap_seq中，并且将原来的snap_seq设置为用户创建的名为”snap-name”的Snapshot的snap_seq。
 4. 每个Snapshot都掌握者一个snap_seq，Image可以看成一个Head Version的Snapshot，每次IO操作对会带上snap_seq发送给Ceph OSD，Ceph OSD会查询该IO操作涉及的object的snap_seq情况。如”object-1″是”image-name”中的一个数据对象，那么初始的snap_seq就”image-name”的snap_seq，当创建一个Snapshot以后，再次对”object-1″进行写操作时会带上新的snap_seq，Ceph接到请求后会先检查”object-1″的Head Version，会发现该写操作所带有的snap_seq大于”object-1″的snap_seq，那么就会对原来的”object-1″克隆一个新的Object Head Version，原来的”object-1″会作为Snapshot，新的Object Head会带上新的snap_seq，也就是librbd之前申请到的。

实验过程可以参考[理解 OpenStack + Ceph （4）：Ceph 的基础数据结构 [Pool, Image, Snapshot, Clone]](http://www.cnblogs.com/sammyliu/p/4843812.html),实验结论摘抄如下：

![ceph-snap-cow](http://7xrnwq.com1.z0.glb.clouddn.com/2017-06-05-ceph-snap-cow.png)



### rbd clone 实现原理

从用户角度来说，clone操作实现了对某个只读snapshot 的image化，即clone之后，可以对这个clone进行读写，snapshot等等，跟一个rbd image一样。
从系统实现来说，也是利用 COW方式实现，clone会将clone与snapshot 的父子关系保存起来,以备IO操作时查找。

![clone-cow](http://7xrnwq.com1.z0.glb.clouddn.com/2017-06-05-clone.png)

实验验证部分参考[理解 OpenStack + Ceph （4）：Ceph 的基础数据结构 [Pool, Image, Snapshot, Clone]](http://www.cnblogs.com/sammyliu/p/4843812.html)。

结论摘抄如下：

![cow-con](http://7xrnwq.com1.z0.glb.clouddn.com/2017-06-05-clone-coe-conclusion.png)


## COW VS ROW

要想了解COW 与ROW，首先需要知道快照的两种类型，一种是全量快照，一种是增量快照。

- 全量快照，顾名思义，就是为源数据卷创建并维护一个完整的镜像卷。
- 增量快照，利用COW 或ROW的方式实现的差量快照。

### COW(COPY-ON-WRITE) 写时复制

在创建快照的时候，仅仅复制一份数据指针表，当读取快照数据时，快照本身没有的话，根据数据指针表，直接读取源数据卷的对应数据块，在更新或写入源数据卷中的数据块时，先将原始数据copy到快照卷中（预留空间），更新快照卷的数据指针表，再将更新数据写入源数据卷。

写时复制的优势在于创建快照速度快（仅复制数据指针表），且占用存储空间少，但因为创建快照后每次写入操作都要进行一次复制才开始写入源数据，所以降低了源数据卷的写性能。
所以COW 适合读多写少的场景，除此之外，如果一个应用容易出现对存储设备的写入热点(只针对某个有限范围内的数据进行写操作),也是比较理想的选择.因为其数据更改都局限在一个范围内（局部性原理）, 对同一份数据的多次写操作只会出现一次写时复制操作。



### ROW(REDIRECT-ON-WRITE)写时重定向

在创建快照的时候，仅仅复制一份数据指针表，当读取快照数据时，快照本身没有的话，根据数据指针表，直接读取源数据卷的对应数据块，在更新或写入源数据卷中的数据块时，将源数据卷数据指针表中的被更新原始数据指针重定向到新的存储空间，所以由此至终, 快照卷的数据指针表和其对应的数据是没有被改变过的。恢复快照的时候,只需要按照快照卷数据指针表来进行寻址就可以完成恢复了.

除了COW的优势外，ROW 因为更新源数据卷只需要一次写操作, 解决了 COW写两次的性能问题. 所以 ROW 最明显的优势就是不会降低源数据卷的写性能。
但ROW 的快照卷数据指针表保存的是源数据卷的原始副本,而源数据卷数据指针表保存的则是更新后的副本,导致在删除快照卷之前需要将快照卷数据指针表指向的数据同步至源数据卷中. 而且当创建了多个快照后, 会产生一个快照链,使原始数据的访问、快照卷和源数据卷数据的追踪以及快照的删除将变得异常复杂且消耗时间。
除此之外,因为源数据卷数据指针指向的数据会很快的被重定向分散, 所以 ROW 另一个主要缺点就是降低了读性能(局部空间原理)。
在传统存储设备上,ROW快照在多次读写后,源数据卷的数据被分散,对于连续读写的性能不如COW. 所以 ROW 比较适合Write-Intensive(写密集) 类型的存储系统. 但是, 在分布式存储设备上, ROW 的连续读写的性能会比 COW 更加好. 一般而言,读写性能的瓶颈都在磁盘上.而分布式存储的特性是数据越是分散到不同的存储设备中, 系统性能越高。所以ROW的源数据卷重定向分散性反而带来了好处。 因此, ROW 逐渐成为了业界分布式存储的主流。







## 参考文章


[SNAPSHOTS](http://docs.ceph.com/docs/master/rbd/rbd-snapshot/)

[解析Ceph: Snapshot](http://www.wzxue.com/%E8%A7%A3%E6%9E%90ceph-snapshot/)

[关于Ceph的snapshot和clone](http://blog.dnsbed.com/archives/1066)

[Ceph Snapshots: Diving into Deep Waters](http://events.linuxfoundation.org/sites/events/files/slides/2017-03-23%20Vault%20Snapshots.pdf)

[理解 OpenStack + Ceph （4）：Ceph 的基础数据结构 [Pool, Image, Snapshot, Clone]](http://www.cnblogs.com/sammyliu/p/4843812.html)

[ ROW/COW 快照技术原理解析](http://blog.csdn.net/jmilk/article/details/65629391)

***END***
