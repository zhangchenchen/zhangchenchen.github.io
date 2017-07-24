---
layout: post
title:  "Openstack-- 最近遇到的几个Openstack问题小结"
date:   2017-07-20 16:59:11
tags: 
  - openstack
  - ceph
---


##  win7虚拟机CPU核数不变问题

问题描述：虚拟机镜像为win7系统，配置为4cpu的时候，发现设备管理器显示为4cpu，但是任务管理器智能识别2cpu，如下图。

![shebei](http://7xrnwq.com1.z0.glb.clouddn.com/2017-07-24-renwu.png)

![renwu](http://7xrnwq.com1.z0.glb.clouddn.com/2017-07-24-shebei.png)

问题解决：刚开始以为是win7系统引导项中限制到2核，更改后问题没有解决，更换镜像后问题依旧，后经过搜索得出答案:首先需要了解kvm的cpu虚拟化原理，可以参考[KVM 介绍（2）：CPU 和内存虚拟化](http://www.cnblogs.com/sammyliu/p/4543597.html),简单来说，一个KVM虚拟机就是一个Linux qemu-kvm进程，内存就是该进程的地址空间的一部分，虚拟机的vcpu作为线程运行在该进程的上下文，逻辑关系如下：

![kvm-topology](http://7xrnwq.com1.z0.glb.clouddn.com/2017-07-24-kvm-topology.jpg)
我们用kvm命令创建虚拟机时有几个概念，
```bash
$ kvm -m 2048 -smp 4,sockets=4,cores=1,threads=1 -drive file=win7_x64_pure 
```
其中，smp 指定为4 ，默认后面什么都不加的话，相当于是sockets=4,cores=1,threads=1，简单解释下socket是cpu的物理单位，core是每个cpu中的物理内核，thread是利用超线程技术实现的一个core虚拟化出的逻辑cpu个数。也就是说，客户机操作系统看到的cpu核数是上述三个数值的乘积，至于使用多socket，还是多core，可以参考[vCPU configuration. Performance impact between virtual sockets and virtual cores?](http://frankdenneman.nl/2013/09/18/vcpu-configuration-performance-impact-between-virtual-sockets-and-virtual-cores/)。
回到之前的问题，openstack默认的max-sockets是4，也就是说只要socket没有特殊指定，且小于等于4，那么逻辑核数就是socket的个数。比如，创建一个4核的kvm虚拟机，那么openstack默认对应kvm命令为kvm -smp 4,sockets=4,cores=1,threads=1。而在某些客户机操作系统会限制物理 CPU （这里即socket）的数目,比如win7操作系统限制2个win，Windows Server 2008 R2 Standard Edition 限制4个。这种情况下，我们就需要更改cpu topology,比如win7,为了实现4核，可以利用以下命令：
```bash
kvm -m 2048 -smp 4,sockets=2,cores=1,threads=2 -drive file=win7_x64_pure 
```
更多cpu topology 的知识，参考[VirtDriverGuestCPUTopology](https://wiki.openstack.org/wiki/VirtDriverGuestCPUTopology)。
回到openstack的话，我们可以给镜像添加合适的 max-sockets来解决此类问题，在win 7的镜像里加上一个属性, hw_max_sockets=2即可。


## 虚拟机调整配额失败

问题描述：如题，对已经启动的虚拟机调整配额，没有反应。
问题解决：找到对应计算节点，查看nova-compute日志发现错误原因，错误日志如下：
![nova-compute-error](http://7xrnwq.com1.z0.glb.clouddn.com/2017-07-24-error-log.png)

无法ssh到目标节点，查阅相关资料后，得出resize命令需要设置nova用户在节点之间的passwordless authentication，解决如下：
 - sudo -u nova ssh-keygen   # 生成nova的密钥
 - ssh-copy-id nova@<serverIP> #复制公钥到目的节点


## ceph 节点资源耗尽

问题描述：公司有个生产环境的云平台，因为前期的滥用，导致存储资源耗费非常严重，已经出现2个计算节点near full，这会导致出现一系列问题，比如虚拟机卡顿，无法访问等问题。
解决方案：
 - 删除不用的虚拟机，镜像等垃圾文件，如果出现无法删除的情况，直接利用rbd命令尝试下，如果还是无法删除，应该是对应osd达到full状态而拒绝客户端操作命令了，只能换个方法。
 - 如果条件允许的话，最好是增加OSD节点，然后再平衡就OK了，不过再平衡的时间太长，对于线上业务是无法接受的。
 - 通过rewight命令手动修改对应osd wight值，比如，对于处于near full/full状态的osd，我们减小其对应的weight值，适当增大有较多剩余空间的osd的weight值。调整的过程中不要一下全部更改，需要调整一下，看下效果，避免因为改动过大出现大范围的再平衡导致时间过长。
 - 在osd 再平衡期间，增加mon-osd-full-ratio/mon osd nearfull ratio值（未验证） 



## 参考文章


[KVM 介绍（2）：CPU 和内存虚拟化](http://www.cnblogs.com/sammyliu/p/4543597.html)

[Openstack Windows7 CPU核数显示不一致](http://www.pystack.org/openstack-windows7-cpu-core-count-display-inconsistencies/)

[vCPU configuration. Performance impact between virtual sockets and virtual cores?](http://frankdenneman.nl/2013/09/18/vcpu-configuration-performance-impact-between-virtual-sockets-and-virtual-cores/)

[Host key verification failed on VM Resize](http://lists.openstack.org/pipermail/openstack-operators/2013-January/002424.html)

[HANDLING A FULL CEPH FILESYSTEM](http://docs.ceph.com/docs/master/cephfs/full/)

[Ceph集群磁盘没有剩余空间的解决方法](http://xiaoquqi.github.io/blog/2015/05/12/ceph-osd-is-full/)

***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***