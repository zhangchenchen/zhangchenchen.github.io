---
layout: post
title:  "openstack系列--swift"
date:   2016-09-02 14:24:25
tags: openstack
---

## 目录结构：

在学习一个新东西的时候，我们其实默认是带着问题去学习的。但当我们真正在卷帙繁多的互联网或者书籍里获取信息时。之前的问题往往就淡化了，反而更多的是跟着资料所给的思路去学习。这样有好处也有坏处，好处是我们不用费心去思考，而坏处也是如此，不用思考带来的后果就是这些信息虽然当时我们理解了，但很难在我们的脑海中建立知识图谱，比较容易遗忘。

所以以后我会在写文章的时候先把问题具象化，从自己问题的角度出发去寻找答案。比如本篇我们学习openstack的对象存储模块swift，出现在脑海中的第一个问题就是 *swift 是做什么用的？* 在解决了这个问题以后，我们知道
是做对象存储的，那么， *它是如何进行存储的？* ，这就是第二个问题。在了解了它的存储机制后，我们还想问的就是 *既然是存储，那么它的安全性机制是怎么做的？数据丢失了怎么办？* 。最后我们就要了解下 *swift 的具体使用* 以及 深入到代码层面 *源码的组织架构* 等。

[swift什么用？ ](#A)

[swift中数据如何存储？ ](#B)

[swift中的安全机制](#C)

[swift 如何使用？](#D)

[swift 源码目录结构](#E)





<a name="A"></a>

## swift 什么用？
我们已经知道swift是openstack中负责对象存储的组件。自然而然的引出另一个问题，什么是对象存储？除了对象存储还有其他什么类型的存储，区别是什么？OK，接下来慢慢解决这些问题。

对象存储，有点类似于hashmap这种数据结构，我们可以通过key来获取/删除对应的value值。对象存储就是将对象数据作为一个逻辑单元存储起来，更多的操作是获取，删除。更新操作频率比较低。相应的，对象存储比较适合于存放静态数据或者更新频率比较低的大容量数据。比如一些多媒体数据，数据备份，虚拟机镜像等。

其他的存储类型还包括块存储，文件存储。

块存储通常提供的原始块设备，就是裸磁盘空间，比如磁盘阵列里面有2块硬盘，然后可以通过划逻辑盘、做Raid、或者LVM（逻辑卷）等种种方式逻辑划分出N个逻辑的硬盘，再采用映射的方式将这几个逻辑盘映射给主机使用。一些对实时性要求比较高的存储我们就得用块存储了，比如数据库。openstack中对应的组件为cinder。

文件存储就比较好理解了，我们常用的Windows操作系统就搭载了一套文件系统，提供文件存储。在分布式环境中，常用NFS协议，通过TCP/IP实现网络化存储，比如我们常用的NFS或者FTP服务器。

<a name="B"></a>

## swift中数据如何存储？

首先看下swift的整体架构：

![swift_arvhitecture](http://7xrnwq.com1.z0.glb.clouddn.com/20160902-swift_architecture.jpg)

整体架构主要分为两部分，访问层（Access Tier）和存储层（Storage Node）。访问层最主要的有两部分，Proxy node 主要负责Http请求的的转发，还有一个负责用户身份的认证（Authentication）。可选的是一个负载均衡设备。

当一个Restful请求到来时，Proxy node 负责接受用户请求并转发给认证服务进行处理，认证通过后再转发给存储层进行数据的操作。在转发给存储层之前，如果启用缓存的话，首先会去缓存服务器检查是否命中，命中就不用去数据层了。

接下来我们看下存储层，存储层由一系列的存储节点组成。为了便于组织以及故障隔离，这些存储节点在物理上做了一些划分，比如根据地理位置的不同划分region,一个 region相当于一个数据中心，每个rigion内部有多个zone，zone可以理解为一组独立的存储节点。一个zone包含多个存储节点（storage node）,一个node里有多个Device(可以理解为磁盘)，一个Device包含多个Partition(可以理解为磁盘中文件系统上的一个目录)。

在每个Storage node 上存储的对象在逻辑上又分为三层：

![swift 对象组织架构](http://7xrnwq.com1.z0.glb.clouddn.com/20160902swift-object-acchitectture.png)

相应的，Storage node 中运行着三种对应的服务：

 - Account Server :Account server 负责Account相关的一些服务，比如包含的Countainer列表，以及Account的元数据等。这些数据存储在一个SQLLite数据库中。
 - Countainer Server : 负责Countainer相关服务，比如包含的object列表，Countainer元数据。也存储在SQLlite数据库中。
 - Object Server : 提供对象数据的存取及相应的元数据服务，以二进制的形式存储在存储节点中，元数据以扩展属性的形式存储其中。因为对应的存储系统必须支持文件的扩展属性。

由以上信息我们得知swift对象最终以二进制的形式存储在存储节点中，存储节点中并没有“路径”的概念，那么它最终是如何与物理位置映射的呢？swift提出了ring的概念，ring其实就是记录了存储对象与物理存储节点的映射关系，object，countainer，account都有与之对应的ring。proxy server接收到http请求时，先查找操作实体（object，countainer，account）对应的ring,根据ring确定他们在对应的服务器集群中的具体位置，并将对应的http请求转发给对应的节点上的server。

此处注意的是，对象寻址并非是渐进式的（即寻找某个object先寻找account，再寻找下面的countainer,再寻找对应的object），而是直接寻找（即寻找object ,直接ring就可以找到，同理寻找某个account，countainer也是如此）。

为什么叫ring呢？其实这跟那个映射算法有关，这个将数据映射在相应服务器集群上某个节点的算法叫一致性hash算法，关于该算法，可以花5分钟的时间了解下[这篇文章](http://blog.csdn.net/cywosp/article/details/23397179)


至于swift 中是如何具体实现该算法的就不详细叙述了。下图为大致架构图：

![openstack-swift-logic-architecture](http://7xrnwq.com1.z0.glb.clouddn.com/20160904-openstack-swift-logic-architecture.png)


<a name="C"></a>

## swift中的安全机制

swift 中为了保证数据在损坏的情况下依然保证高可用，采用了增加副本的策略，每一个对象都会有若干个备份的副本（默认是三个），且存储在不同的zone里，这样，当某一个zone的对象数据不可用时，还可以启用其他zone里的副本。这样为了解决数据一致性的问题，swift开启了以下三个服务：

 - Replicator: 负责检查数据与相应副本是否一致。如果出现数据不一致的情况，会将过时的数据更新，并负责将标记为删除的数据从磁盘上删除。

 - Auditor: 持续扫描磁盘检查Acount,Container和object数据的完整性，如果发现数据损坏，就会对相应的数据隔离，然后通过Replicator从其他节点上获取副本以恢复。

 - Updator: 在创建一个object的时候，需要对应的更新该object所在的container列表，以及再上层的account 列表，同理，创建container的时候也是如此。有时候这些更新操作因为相应的server繁忙而更新失败，swift会使用Updator服务继续处理这些失败的更新操作。


<a name="D"></a>

## swift的使用

swift的使用跟其他openstack的组件使用一样都是通过restful API来提供服务。swift API主要提供了以下几种功能：
 - 对象存储 单个对象默认最大值是5GB，可以用户自己配置。
 - 超过最大值对象的数据可以通过中间件进行上传，存储
 - 对象压缩
 - 对象删除

swift API 的执行过程大致如下：swiftClient 将用户的命令转换为标准HTTP请求（如果是用户请求本身就是HTTP请求那么没有这一步）;paste deploy将请求路由到proxy-server WSGI Application;根据请求内容调用对应的Controller(AccountController,ContainerController或者ObjectController),该controller会将请求转发到特定的存储节点上的WSGI Server(Account Server,Container Server或Object Server);这些server接收到http请求并处理。

<a name="E"></a>

## swift 源码目录结构

![swift source code](http://7xrnwq.com1.z0.glb.clouddn.com/20160904-openstack-swift-sourcecode.jpg)

 - bin: 主要是一些启动脚本，工具脚本。比如proxy-server负责启动proxy server，swift-ring-builder 用来创建ring。
 - swift：swift的核心代码。其子目录account,container,obj,proxy分别对应相应的服务的具体实现。common子目录是被多个组件共用的公共代码。
 - etc:配置文件模板


## 参考文章

[openstack 设计与实现](https://book.douban.com/subject/26374647/)

[五分钟了解一致性hash](http://blog.csdn.net/cywosp/article/details/23397179)

[swift github](https://github.com/openstack/swift)

***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***
