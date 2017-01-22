---
layout: post
title:  "openstack系列--Nova"
date:   2016-08-22 14:39:25
tags: openstack
---

## 目录结构：

[概述 ](#A)

[体系架构 ](#B)

[VM实例的生命周期管理](#C)

[动态迁移实例](#D)

[代码结构与部署](#E)




<a name="A"></a>

## 概述 

 > OpenStack Nova provides a cloud computing fabric controller, supporting a wide variety of virtualization technologies, including KVM, Xen, LXC, VMware, and more. In addition to its native API, it includes compatibility with the commonly encountered Amazon EC2 and S3 APIs.

也就是说，Nova 是一个控制器，它支持利用各种虚拟化技术（KVM,XEN等）完成云计算平台的CPU,内存，硬盘的虚拟化。云中的实例instance生命周期的所有活动都是由Nova控制。Nova自身并没有提供任何虚拟化能力，它使用libvirt API来与被支持的Hypervisors交互。Nova 通过一个与Amazon Web Services（AWS）EC2 API，S3 API兼容的web services API来对外提供服务。




<a name="B"></a>

## 体系架构

首先放一张 Openstack 的整体概念结构图：

![Conceptual Architecture](http://7xrnwq.com1.z0.glb.clouddn.com/20160822openstack-concept-architeucture.jpg)

第二张图是逻辑结构图(图是13年的，现在的Network Service已经变更为Neutron)

![openstack architecture](http://7xrnwq.com1.z0.glb.clouddn.com/20160822openstack-architecture.jpg)

图中可以看出，Nova处于Openstack的一个核心位置，其他组件都为 Nova 提供支持，Glance 为 VM 提供 image ，Cinder 和 Swift 分别为 VM 提供块存储和对象存储，Neutron 为 VM 提供网络连接。

接下来一张就是Nova的架构图：

![nova architecture](http://7xrnwq.com1.z0.glb.clouddn.com/20160822-nova-architecture.png)

这些组件以子服务（后台 deamon 进程）的形式运行。总的来说，nova的各个组件是以数据库和队列为中心进行通信的，下面对其中的几个组件做一个简单的介绍：

 - nova-api: 接收和响应客户的 API 调用。 除了提供 OpenStack 自己的API，nova-api 还支持 Amazon EC2 API。

 - nova-scheduler: 虚机调度服务，负责决定在哪个计算节点上运行虚机 。

 - nova-compute: 管理虚机的核心服务，通过调用 Hypervisor API 实现虚机生命周期管理 。

 - Hypervisor: 计算节点上跑的虚拟化管理程序，虚机管理最底层的程序。 不同虚拟化技术提供自己的 Hypervisor。 常用的 Hypervisor 有 KVM，Xen， VMWare 等 。

 - nova-conductor: nova-compute 经常需要更新数据库，比如更新虚机的状态。 出于安全性和伸缩性的考虑，nova-compute 并不会直接访问数据库，而是将这个任务委托给 nova-conductor。

 - Database: Nova 会有一些数据需要存放到数据库中，一般使用 MySQL。 数据库安装在控制节点上。 Nova 使用命名为 “nova” 的数据库。

  - Message Queue: 为解耦各个子服务，Nova 通过 Message Queue 作为子服务的信息中转站。OpenStack 默认是用 RabbitMQ 作为 Message Queue。




<a name="C"></a>

## VM实例的生命周期管理

![VM实例的生命周期](http://7xrnwq.com1.z0.glb.clouddn.com/20160822-openstack-instance-lifetime.png)

有的操作功能比较类似，也有各自的适用场景，简单介绍下上述几个重要的操作：

 - 常规操作： 常规操作中，Launch、Start、Reboot、Shut Off 和 Terminate 都很好理解。 下面几个操作重点回顾一下：

    - Resize: 通过应用不同的 flavor 调整分配给 instance 的资源。

    - Lock/Unlock:  可以防止对 instance 的误操作。 

    - Pause/Suspend/Resume: 暂停当前 instance，并在以后恢复。 Pause 和 Suspend 的区别在于 Pause 将 instance 的运行状态保存在计算节点的内存中，而 Suspend 保存在磁盘上。 Pause 的优点是 Resume 的速度比 Suspend 快；缺点是如果计算节点重启，内存数据丢失，就无法 Resume 了，而 Suspend 则没有这个问题。

    - Snapshot:  备份 instance 到 Glance。产生的 image 可用于故障恢复，或者以此为模板部署新的 instance。 

 -  故障处理: 故障处理有两种场景：计划内和计划外。计划内是指提前安排时间窗口做的维护工作，比如服务器定期的微码升级，添加更换硬件等。 计划外是指发生了没有预料到的突发故障，比如强行关机造成 OS 系统文件损坏，服务器掉电，硬件故障等。

    计划内故障处理:对于计划内的故障处理，可以在维护窗口中将 instance 迁移到其他计算节点。 涉及如下操作： 

       - Migrate: 将 instance 迁移到其他计算节点。 迁移之前，instance 会被 Shut Off，支持共享存储和非共享存储。 
       - Live Migrate： 与 Migrate 不同，Live Migrate 能不停机在线地迁移 instance，保证了业务的连续性。也支持共享存储和非共享存储（Block Migration）。
       - Shelve/Unshelve： Shelve 将 instance 保存到 Glance 上，之后可通过 Unshelve 重新部署。 Shelve 操作成功后，instance 会从原来的计算节点上删除。 Unshelve 会重新选择节点部署，可能不是原节点。

    计划外故障处理： 划外的故障按照影响的范围又分为两类：Instance 故障和计算节点故障 。

       Instance 故障:Instance 故障只限于某一个 instance 的操作系统层面，系统无法正常启动。 可以使用如下操作修复 instance。

       - Rescue/Unrescue: 用指定的启动盘启动，进入 Rescue 模式，修复受损的系统盘。成功修复后，通过 Unrescue 正常启动 instance。 

       - Rebuild: 如果 Rescue 无法修复，则只能通过 Rebuild 从已有的备份恢复。Instance 的备份是通过 snapshot 创建的，所以需要有备份策略定期备份。 

       计算节点故障: Instance 故障的影响范围局限在特定的instance，计算节点本身是正常工作的。如果计算节点发生故障，OpenStack 则无法与节点的 nova-compute 通信，其上运行的所有 instance 都会受到影响。这个时候，只能通过 Evacuate 操作在其他正常节点上重建 Instance。

       - Evacuate:  利用共享存储上 Instance 的镜像文件在其他计算节点上重建 Instance。 所以提前规划共享存储是关键。


<a name="D"></a>

## 动态迁移实例

*注意*：以下内容大多摘自[每天五分钟玩转openstack](https://www.ibm.com/developerworks/community/blogs/132cfa78-44b0-4376-85d0-d3096cd30d3f?sortby=0&maxresults=15&page=2&lang=en)，强烈推荐，如果觉得TLDR,以下为部分节选。

接下来以一个instance 动态迁移的示例来看下Nova的具体执行流程。

Migrate 操作会先将 instance 停掉，也就是所谓的“冷迁移”。而 Live Migrate 是“热迁移”，也叫“在线迁移”，instance不会停机。

Live Migrate 分两种：

 - 源和目标节点没有共享存储，instance 在迁移的时候需要将其镜像文件从源节点传到目标节点，这叫做 Block Migration（块迁移）
 
 - 源和目标节点共享存储，instance 的镜像文件不需要迁移，只需要将 instance 的状态迁移到目标节点。

源和目标节点需要满足一些条件才能支持 Live Migration：
 
 - 源和目标节点的 CPU 类型要一致。

 - 源和目标节点的 Libvirt 版本要一致。

 - 源和目标节点能相互识别对方的主机名称，比如可以在 /etc/hosts 中加入对方的条目。

 - 在源和目标节点的 /etc/nova/nova.conf 中指明在线迁移时使用 TCP 协议。

 - Instance 使用 config driver 保存其 metadata。在 Block Migration 过程中，该 config driver 也需要迁移到目标节点。由于目前 libvirt 只支持迁移 vfat 类型的 config driver，所以必须在 /etc/nova/nova.conf 中明确指明 launch instance 时创建 vfat 类型的 config driver。

 - 源和目标节点的 Libvirt TCP 远程监听服务得打开

接下来以非共享存储Block Migration 为例简述下Nova的工作流程：

![block migration](http://7xrnwq.com1.z0.glb.clouddn.com/20160823nova-process.jpg)

1. 向 nova-api 发送请求：通过dashboard 发送live migrate消息，指定源节点，目标节点。

2. nova-api 发送消息:nova-api 向 Messaging（RabbitMQ）发送了一条消息：“Live Migrate 这个 Instance” 源代码在 /opt/stack/nova/nova/compute/api.py，方法是 live_migrate。

3. nova-compute 执行操作：
   
   -  目标节点执行迁移前的准备工作，首先将instance的数据迁移过来，主要包括镜像文件、虚拟网络等资源，日志在 devstack-controller:/opt/stack/logs/n-cpu.log。
   
   - 源节点启动迁移操作，暂停 instance。

   - 在目标节点上 Resume instance

   - 在源节点上执行迁移的后处理工作，删除 instance。

   - 在目标节点上执行迁移的后处理工作，创建 XML，在 Hypervisor 中定义 instance，使之下次能够正常启动。



<a name="E"></a>

## 代码结构与部署

 *未完待续*


## 参考文章

[大家谈OpenStack-Nova组件理解](http://www.aboutyun.com/thread-6733-1-1.html)

[OpenStack Grizzly Architecture ](http://solinea.com/blog/openstack-grizzly-architecture-revisited)

[OpenStack Hacker养成指南](http://www.openstack.cn/?p=392)

[每天5分钟玩转 OpenStack](https://www.ibm.com/developerworks/community/blogs/132cfa78-44b0-4376-85d0-d3096cd30d3f?sortby=0&maxresults=15&page=3&lang=en)

[openstack doc](http://docs.openstack.org/mitaka/networking-guide/intro-os-networking-overview.html)

[openstack 设计与实现](https://www.amazon.cn/Open-Stack%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0-%E8%8B%B1%E7%89%B9%E5%B0%94%E5%BC%80%E6%BA%90%E6%8A%80%E6%9C%AF%E4%B8%AD%E5%BF%83/dp/B00WG4WJ9I/ref=sr_1_1?ie=UTF8&qid=1471599232&sr=8-1&keywords=openstack+%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0)


***END***
