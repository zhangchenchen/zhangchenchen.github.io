---
layout: post
title:  "Hadoop-- Hadoop学习之HDFS"
date:   2018-01-30 14:57:10
tags: 
  - Hadoop
  - HDFS
  - Bigdata
---


## 概述

最近因为项目需要将重心转移到大数据架构这一块，所以将大数据的内容大致过一下，在此记录，内容大部分来自官方文档与博客，主要基于Hadoop 的最新stable版本（2.9）。
学习大数据，立马想到的就是Hadoop，Hadoop是一个开源的可依赖，可扩展的分布式计算框架，采用普通PC机集群完成对大量数据的存储，计算。这个项目主要包括四个模块：
- Hadoop Common: Hadoop的Common utilities模块。
- Hadoop Distributed File System (HDFS): 一个支持高可用的分布式文件系统。
- Hadoop YARN: 任务调度和资源管理框架。
- Hadoop MapReduce: 基于YARN的对大量数据并行处理的计算模型。
除了以上四个主要模块之外，Hadoop生态还有很多其他的的系统，以后再慢慢写，这里先写Hadoop生态中最重要的一个组件HDFS。

先看下Hadoop里的服务角色：

![hadoop-server-role](http://7xrnwq.com1.z0.glb.clouddn.com/20180201160125-hadoop-role.jpg)

Hadoop主要的任务部署分为3个部分，分别是：Client机器，主节点和从节点。主节点主要负责Hadoop两个关键功能模块HDFS、Map Reduce的监督。Job Tracker使用Map Reduce进行监控和调度数据的并行处理，namenode则负责HDFS监视和调度。从节点负责了机器运行的绝大部分，担当所有数据储存和指令计算的苦差。每个从节点既扮演着数据节点的角色又充当与他们主节点通信的守护进程。守护进程隶属于Job Tracker，数据节点则归属于名称节点。Client负责把数据加载到集群中，递交给Map Reduce做数据处理工作，并在工作结束后取回或者查看结果。

再看下典型的围绕hadoop 的workflow：

![hadoop-workflow](http://7xrnwq.com1.z0.glb.clouddn.com/20180201160825hadoop-workflow.jpg)

## HDFS Architecture

### 架构
HDFS的整体设计架构就是master/slave，namenode负责元数据的存取，数据管理等功能，datanode负责数据存储。如下：

![hdfs-arch](http://7xrnwq.com1.z0.glb.clouddn.com/20180201154432-hdfs-arch.jpg)
### datanode工作原理

数据的存储方式与ceph类似，默认都是三副本，先将大数据文件切割成固定大小（比如128M）的block，然后将这些block存放到三个不同的datanode中，namenode会记录对应文件block以及block所在位置。
举例一个datanode存储过程:client先做文件切割，并提交存储文件命令给namenode，namenode有一个rack awareness的功能，简单点说就是会将数据存储到不同机架上以避免机架故障（电源故障等），如果是三副本的话，首先client会写入block到某一节点A，然后另一机架中的节点B 会从A复制该数据,再然后同一机架内的另一个节点C会从B复制一份数据。这样一个pipeline既保证了数据的容灾，也能减小数据传输的延迟。

![datanode](http://7xrnwq.com1.z0.glb.clouddn.com/20180201161950-data-node.jpg)

### namenode工作原理

理解namenode，首先要理解两个文件。一个是 Edits文件，一个是FsImage映像文件，这两个文件就包含整个HDFS集群的元数据，而namenode的最大任务就是维护这两个文件。
- Edits文件：NameNode在本地操作系统的文件都会保存在Edits日志文件中。也就是说当文件系统中的任何元数据产生操作时，都会记录在Edits日志文件中。eg：在HDFS上创建一个文件，NameNode就会在Edits中插入一条记录。同样如果修改或者删除等操作，也会在Edits日志文件中新增一条数据。
- FsImage映像文件：整个文件系统的名字空间，包括数据块到文件的映射，文件的属性等等，都存储在一个称为FsImage的文件中，这个文件也是放在NameNode所在的文件系统中。（注意到该文件中并没有blockmap,描述数据块Block与DataNode节点之间的对应关系的文件，这是因为每个DataNode已经持有属于自己管理的Block集合，每次blockreport都会将所有DataNode的Block集合汇总后即可构造出完整BlocksMap，所以不用持久化。）

为什么会引入这两个文件呢，因为在HDFS的整个运行期里，所有元数据均在NameNode的内存集中管理，但是由于内存易失特性，一旦出现进程退出、宕机等异常情况，所有元数据都会丢失，为了更好的容错能力，NameNode会周期进行Checkpoint，将其中的一部分元数据（文件系统的目录树Namespace）刷到持久化设备上，即二进制文件FSImage，这样的话即使NameNode出现异常也能从持久化设备上恢复元数据。但是仅周期进行Checkpoint仍然无法保证所有数据的可靠，如前次Checkpoint之后写入的数据依然存在丢失的问题，所以将两次Checkpoint之间对Namespace写操作实时写入EditLog文件，通过这种方式可以保证HDFS元数据的绝对安全可靠。

首先看一下namenode启动时做了哪些操作（下文部分直接引用自[该博客](http://blog.csdn.net/mmd0308/article/details/74674524)）：

![namenode](http://7xrnwq.com1.z0.glb.clouddn.com/20180201163924-namenode.jpg)

接下来看看check point的时候做了哪些操作,这个时候就要引入Secondary NameNode了，NameNode主要是存储文件的metadata，运行时所有数据都保存在内存中，这个的HDFS可存储的文件受限于NameNode的内存。而Secondary NameNode可以看做是NameNode的灾备（并非HA），它会定时与NameNode进行同步，定期的将fsimage映像文件和Edits日志文件进行合并，并将合并后的传入给NameNode，替换其镜像，并清空编辑日志。如果NameNode失效，需要手动的将其设置成namenode主机。
checkpoint的时间默认是3600秒（可配置），当Edits日志文件超过最大值时也会进行check point。checkpoint大致如下：
- NameNode通知Secondary NameNode进行checkpoint。
- Secondary NameNode通知NameNode切换edits日志文件，使用一个空的。
- Secondary NameNode通过Http获取NmaeNode上的fsimage映像文件和切换前的edits日志文件。
- Secondary NameNode在内容中合并fsimage和Edits文件。
- Secondary NameNode将合并之后的fsimage文件发送给NameNode。
- NameNode用Secondary NameNode 传来的fsImage文件替换原先的fsImage文件

## HDFS 使用


### 常用操作

HDFS本质是一个文件系统，所以跟Linux 的文件系统使用类似，也是增删改查，不过这里的“改”不是update，而是truncate，也就是文件截断，HDFS 不支持update操作，这与它读多写少的特性相对应。
对HDFS的操作可以通过shell，web 界面，libhdfs (C API)或WebHDFS (REST API)来操作。HDFS里的文件跟linux 文件类似，也有own user，group，也有读写执行权限的划分，不过这里的可执行权限只对目录有用，意思是是否有权限对该目录的子目录或文件有可读权限。每一个文件的操作命令都会进行Permission Checks，不通过则fail。
HDFS的具体shell命令略，可参考[FileSystemShell](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/FileSystemShell.html)。

### Quotas 设置

HDFS支持 administrator对quota的设置，包括：
- name quota: 指定目录下文件或者目录的数量限制。
- space quota: 指定目录下文件的占用空间限制。
- Storage Type Quotas：指定目录下Storage Type的限制。

administrator可以通过命令行或其他的方式进行设置。

### 透明加密

透明加密主要是防止application与HDFS之间进行端到端的数据传输时的数据安全，不需要更改user application的任何代码。
详细参考[TransparentEncryption](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/TransparentEncryption.html)。

## HDFS HA

这里的HA主要是指namenode的HA。hadoop文档提供了两种方法，本质其实是一样的，因为namenode主要依靠FsImage和EditLog两个文件管理DataNode的数据，要想保证新的namenode能随时替换，就要保证这两个文件的一致性，FsImage是存储在磁盘上还好说，EditLog在内存里随时变化，就要保证两个namenode的EditLog文件是一样的。两个方案如下：
- QJM：the Quorum Journal Manager，这种方案是通过JournalNode共享EditLog的数据，使用的是Paxos算法（zookeeper就是使用的这种算法），保证活跃的NameNode与备份的NameNode之间EditLog日志一致,配合zookeeper可以实现自动切换,推荐使用。
- NFS：Network File System 或 Conventional Shared Storage，传统共享存储，其实就是在服务器挂载一个网络存储（比如NAS），Active NameNode将EditLog的变化写到NFS，备份NameNode检查到修改就读取过来，是两个NameNode数据一致,缺点是如果namenode或者standby namenode与NFS磁盘之间的网络出了问题，HA即失效。

![HDFS-HA](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/2018-2-2-HDFS-HA.jpg)

以上是QJM HA的典型的结构图。集群中共有两个namenode(简称NN)，其中只有一个是active状态，另一个是standby状态。active 的NN负责响应DN(datanode)的请求，为了最快的切换为active状态，standby状态的NN同样也连接到所有的datenode上获取最新的块信息(blockmap)。
active NN会把元数据的修改(edit log)发送到多数的journal节点上(2n+1个journal节点，至少写到n+1个上)，standby NN从journal节点上读取edit log，并实时的合并到自己的namespace中。另外standby NN连接所有DN，实时的获取最新的blockmap。这样，一旦active的NN出现故障，standby NN可以立即切换为active NN.

具体配置参考[HADOOP(二):HDFS 高可用原理](http://www.bijishequ.com/detail/373246)

## HDFS 认证与授权

为了确保数据安全，认证授权这一块hadoop也下了一番功夫，一般来说，需要经历一下几个阶段：
- 认证
- proxy user
- service level Authorization
- 第三方权限控制Ranger
- Hadoop POSIX ACLs

![hadoop-auth](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/2018-02-02Hadoop%20Range.png)

认证部分有两种方式，simple和kerberos，simple不做任何处理，会由操作系统层获取用户，客户端可以通过设置环境变量HADOOP_USER_NAME来伪装用户。kerberos认证对集群里的所有机器都分发了keytab，使得集群机器进程之间不能随便访问。
代理部分：当客户端访问hadoop时，并不想以当前进程用户去调用，上层应用一般有自己一套用户管理体系，所以hadoop提供代理机制，让进程用户可以代理登录用户提交请求。然而，如果对代理用户不加以控制的话，那权限便相对于无限放大，比如代理超级用户：hdfs，yarn等，所以对于进程用户会设置可在哪些主机提交请求和代理哪些用户组成员。 
权限控制首先经过Service Level Authorization，检测服务级别权限。 
比如哪些用户可以连接namenode，resourcemanager，属于服务级别的acl控制。参考[Hadoop服务层授权控制](https://www.iteblog.com/archives/983.html)
接着由Ranger进行目录，队列等资源的权限管控, 属于更细粒度的权限控制。如果ranger没有策略控制，则进入原始HDFS文件系统权限或者MR权限控制。


## 参考文章

[Welcome to Apache Hadoop](http://hadoop.apache.org/)

[HDFS Users Guide](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HdfsUserGuide.html)

[深入理解Hadoop集群和网络](http://pangjiuzala.github.io/2015/08/02/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3Hadoop%E9%9B%86%E7%BE%A4%E5%92%8C%E7%BD%91%E7%BB%9C/)

[Hadoop2.7.3 HA高可靠性集群搭建](http://www.cnblogs.com/zhangmingcheng/p/6406792.html)

[Hadoop之HDFS分布式文件系统NameNode及Secondary NameNode详解](http://blog.csdn.net/mmd0308/article/details/74674524)

[Hadoop 访问管理](http://komi.leanote.com/post/Hadoop-Ranger%E6%9D%83%E9%99%90%E6%B5%81%E7%A8%8B)


***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***