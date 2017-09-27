---
layout: post
title:  "Kubernetes-- 关于ETCD && 服务发现的一些记录"
date:   2017-08-05 14:28:00
tags: 
  - kubernetes
  - ETCD
---


##  ETCD简介

ETCD是一个可靠的键值对分布式存储系统，适用场景是用来存储那些写少读多，结构简单，但又比较重要的数据，这里的比较重要是指要保证数据的高可用，一致性。第一个想到的自然就是配置数据的存储与管理。当然，也可以存储其他满足上述要求的数据，比如在k8s中，ETCD会用来存储k8s各items(pods, rc, rs, deployment等)的状态。随着微服务架构的日益火热，ETCD也常作为服务发现的组件来使用。
ETCD 有以下几个特性：

- 使用简单：友好的API(grpc)
- 安全：支持TLS双向通信
- 快速：支持10,000 写/秒
- 可靠： 采用raft协议作分布式选举


## ETCD 安装

因为是go写的，直接下载二进制文件执行即可，如果想要尝试最新版的，可以去下载源码，自己build,参考[Download and build](https://coreos.com/etcd/docs/latest/dl_build.html)。
如果是构造一个cluster，也比较简单，参考[Demo](https://coreos.com/etcd/docs/latest/demo.html)

简单记录下常用的启动参数：

- name：节点名称，默认为 default，在集群中应该保持唯一，一般使用 hostname
- data-dir：服务运行数据保存的路径，默认为 ${name}.etcd
- snapshot-count：指定有多少事务（transaction）被提交时，触发截取快照保存到磁盘
- heartbeat-interval：leader 多久发送一次心跳到 followers。默认值是 100ms
- eletion-timeout：重新投票的超时时间，如果 follower在该时间间隔没有收到心跳包，会触发重新投票，默认为 1000 ms
- listen-peer-urls:和同伴通信的地址，比如 http://ip:2380 ，如果有多个，使用逗号分隔。需要所有节点都能够访问， 所以不要使用 localhost！
- listen-client-urls: 对外提供服务的地址：比如 http://ip:2379,http://127.0.0.1:2379
，客户端会连接到这里和 etcd 交互
- advertise-client-urls: 对外公告的该节点客户端监听地址，这个值会告诉集群中其他节点
- initial-advertise-peer-urls: 该节点同伴监听地址，这个值会告诉集群中其他节点
- initial-cluster: 集群中所有节点的信息，格式为 node1=http://ip1:2380,node2=http://ip2:2380,...... 注意：这里的 node1是节点的--name指定的名字；后面的 ip1:2380是 --initial-advertise-peer-urls
指定的值。
- initial-cluster-state：新建集群的时候，这个值为 new，假如已经存在的集群，这个值为 existing。
- initial-cluster-token：创建集群的 token，这个值每个集群保持唯一。这样的话，如果你要重新创建集群，即使配置和之前一样，也会再次生成新的集群和节点uuid；否则会导致多个集群之间的冲突，造成未知的错误。

当然，还有比如tls设置，使用etcd discovery/dns discovery 进行etcd的启动等，配置详情见官网。

## ETCD 使用

一般通过两种方式，rest api或者通过命令行（本质也是rest api）,下面简单介绍下这两种方式。在介绍前，先要弄懂几个概念：

- member： 指一个 etcd 实例。member 运行在每个 node 上，并向这一 node上的其它应用程序提供服务。
- Cluster： Cluster 由多个 member 组成。每个 member 中的 node 遵循 raft共识协议来复制日志。Cluster 接收来自 member的提案消息，将其提交并存储于本地磁盘。
- Peer： 同一 Cluster 中的其它 member。

### 命令行方式示例

```bash
$ etcdctl --endpoints=$ENDPOINTS put foo "Hello World!"  # 写入操作
$ etcdctl --endpoints=$ENDPOINTS get foo  # 读取操作
$ etcdctl --endpoints=$ENDPOINTS --write-out="json" get foo  # 以json的方式输出
$ etcdctl --endpoints=$ENDPOINTS get web --prefix  # 获取所有前缀是web 的key的value
$ etcdctl --endpoints=$ENDPOINTS del key  # 删除
$ etcdctl --endpoints=$ENDPOINTS txn --interactive  # 将多个命令封装在一个事务中

compares:
value("user1") = "bad"      

success requests (get, put, delete):
del user1  

failure requests (get, put, delete):
put user1 good
$ etcdctl --endpoints=$ENDPOINTS watch stock1  # 监视某个值
$ etcdctl --endpoints=$ENDPOINTS lease grant 300 #设定租约
# lease 2be7547fbc6a5afa granted with TTL(300s)

$ etcdctl --endpoints=$ENDPOINTS put sample value --lease=2be7547fbc6a5afa # 绑定租约
$ etcdctl --endpoints=$ENDPOINTS get sample # 租约期限内可以获取值

$ etcdctl --endpoints=$ENDPOINTS lease keep-alive 2be7547fbc6a5afa # 维持租约
$ etcdctl --endpoints=$ENDPOINTS lease revoke 2be7547fbc6a5afa  # 撤销租约
# or after 300 seconds
$ etcdctl --endpoints=$ENDPOINTS get sample # 撤销租约或者300s后获取不到值
$ etcdctl --write-out=table --endpoints=$ENDPOINTS endpoint status # 集群状态
$ etcdctl --endpoints=$ENDPOINTS endpoint health
$ etcdctl --endpoints=$ENDPOINTS snapshot save my.db # 快照

```

### rest api 示例

```bash 
curl http://127.0.0.1:2379/v2/keys/message # 获取某个key的value
{
    "action": "get",
    "node": {
        "createdIndex": 2,
        "key": "/message",
        "modifiedIndex": 2,
        "value": "Hello world"
    }
}

curl http://127.0.0.1:2379/v2/keys/message -XPUT -d value="Hello etcd" # 更改 
{
    "action": "set",
    "node": {
        "createdIndex": 3,
        "key": "/message",
        "modifiedIndex": 3,
        "value": "Hello etcd"
    },
    "prevNode": {
        "createdIndex": 2,
        "key": "/message",
        "value": "Hello world",
        "modifiedIndex": 2
    }
}

curl http://127.0.0.1:2379/v2/keys/message -XDELETE  #删除 
{
    "action": "delete",
    "node": {
        "createdIndex": 3,
        "key": "/message",
        "modifiedIndex": 4
    },
    "prevNode": {
        "key": "/message",
        "value": "Hello etcd",
        "modifiedIndex": 3,
        "createdIndex": 3
    }
}

```


## 架构 & 原理介绍

![etcd-arch](http://7xrnwq.com1.z0.glb.clouddn.com/etcd-arch-20170927110531.jpg)
etcd主要分为四个部分:

- HTTP Server： 用于处理用户发送的API请求以及其它etcd节点的同步与心跳信息请求。
- Store：用于处理etcd支持的各类功能的事务，包括数据索引、节点状态变更、监控与反馈、事件处理与执行等等，是etcd对用户提供的大多数API功能的具体实现。
- Raft：Raft强一致性算法的具体实现，是etcd的核心。
- WAL：Write Ahead Log（预写式日志），是etcd的数据存储方式。除了在内存中存有所有数据的状态以及节点的索引以外，etcd就通过WAL进行持久化存储。WAL中，所有的数据提交前都会事先记录日志。Snapshot是为了防止数据过多而进行的状态快照；Entry表示存储的具体日志内容。
通常，一个用户的请求发送过来，会经由HTTP Server转发给Store进行具体的事务处理，如果涉及到节点的修改，则交给Raft模块进行状态的变更、日志的记录，然后再同步给别的etcd节点以确认数据提交，最后进行数据的提交，再次同步。


关于集群状态机的转变可以参考[CoreOS 实战：剖析 etcd](http://www.infoq.com/cn/articles/coreos-analyse-etcd)

## 服务发现工具对比

引自[服务发现：Zookeeper vs etcd vs Consul](http://dockone.io/article/667)：

>>>所有这些工具都是基于相似的原则和架构，它们在节点上运行，需要仲裁来运行，并且都是强一致性的，都提供某种形式的键/值对存储。
Zookeeper是其中最老态龙钟的一个，使用年限显示出了其复杂性、资源利用和尽力达成的目标，它是为了与我们评估的其他工具所处的不同时代而设计的（即使它不是老得太多）。
etcd、Registrator和Confd是一个非常简单但非常强大的组合，可以解决大部分问题，如果不是全部满足服务发现需要的话。它还展示了我们可以通过组合非常简单和特定的工具来获得强大的服务发现能力，它们中的每一个都执行一个非常具体的任务，通过精心设计的API进行通讯，具备相对自治工作的能力，从架构和功能途径方面都是微服务方式。
Consul的不同之处在于无需第三方工具就可以原生支持多数据中心和健康检查，这并不意味着使用第三方工具不好。实际上，在这篇博客里我们通过选择那些表现更佳同时不会引入不必要的功能的的工具，尽力组合不同的工具。使用正确的工具可以获得最好的结果。如果工具引入了工作不需要的特性，那么工作效率反而会下降，另一方面，如果工具没有提供工作所需要的特性也是没有用的。Consul很好地权衡了权重，用尽量少的东西很好的达成了目标。
Consul使用gossip来传播集群信息的方式，使其比etcd更易于搭建，特别是对于大的数据中心。将存储数据作为服务的能力使其比etcd仅仅只有健/值对存储的特性更加完整、更有用（即使Consul也有该选项）。虽然我们可以在etcd中通过插入多个键来达成相同的目标，Consul的服务实现了一个更紧凑的结果，通常只需要一次查询就可以获得与服务相关的所有数据。除此之外，Registrator很好地实现了Consul的两个协议，使其合二为一，特别是添加Consul-template到了拼图中。Consul的Web UI更是锦上添花般地提供了服务和健康检查的可视化途径。




## 参考文章


[etcd 使用入门](http://www.liuhaihua.cn/archives/404914.html)

[etcd API](https://coreos.com/etcd/docs/latest/v2/api.html)

[Etcd Tutorial- The Ultimate Reliable Key Value Storage for Networks](https://poweruphosting.com/blog/etcd-tutorial/)

[服务发现比较:Consul vs Zookeeper vs Etcd vs Eureka](https://luyiisme.github.io/2017/04/22/spring-cloud-service-discovery-products/)

[CoreOS 实战：剖析 etcd](http://www.infoq.com/cn/articles/coreos-analyse-etcd)

[深入学习Etcd](http://lihaoquan.me/2016/6/24/learning-etcd-1.html)

[etcd：从应用场景到实现原理的全方位解读](http://www.infoq.com/cn/articles/etcd-interpretation-application-scenario-implement-principle)


***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***