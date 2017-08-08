---
layout: post
title:  "Openstack-- openstack 中的资源分离策略"
date:   2017-08-08 13:54:20
tags: 
  - openstack
---


## 为什么做资源的分离

openstack 作为一个云计算框架，需要统筹计算，存储，网络等资源，本身就已经够复杂了。如果是在一个非常大的环境中，节点众多，且复杂多样，这个时候就有必要根据这些节点的差异做一些逻辑上的分离，以达到不同资源的区分，这样，做水平扩展的时候也可以根据这个逻辑分区针对性的进行扩展。

## openstack 中的资源分离策略

openstack 的资源分离策略一定程度上借鉴了AWS的策略，整体来说，如下：

![resource-segeration](http://oeptotikb.bkt.clouddn.com/2017-08-08-rc-se.png)

### Infrastructure segregation

主要是理解Regions, cells, Host aggregates, Availability zones这几个概念。

- Regions:借鉴自AWS，更像是一个地理上的概念（比如北京的数据中心可以作为一个region,南京的数据中心作为另一个region），每个region有自己独立的endpoint，regions之间完全隔离，不同regions之间可以共享keystone/dashboard。openstack默认新建一个region，即RegionOne，如果还想建立第二个Region，可以利用 keystone endpoint­create 命令添加。
![region](http://oeptotikb.bkt.clouddn.com/2017-08-08-region1.png)


- Cells:cell主要是为了解决openstack 的扩展性以及规模瓶颈而引进的概念。当openstack达到一定规模后，依赖比较强的database以及AMQP便成为整个系统的瓶颈，引入cell后，每个cell有自己独立的database和AMQP。公司云平台只是一个小云，没有引入，所以不去深究。cell目前是有v1，v2两版，实现方式差异比较大，感兴趣可以参考[Nova Cells V2如何帮助OpenStack集群突破性能瓶颈？](https://www.ustack.com/news/what-is-nova-cells-v2/?utm_source=tuicool&utm_medium=referral)。
- Host aggregates && Availability zones：这两个概念有共通点，且要相互配合使用，都用来表示一组节点的集合，简单说，AZ(Availability zone)是一个面向用户的概念，(这里只讨论nova 范畴的AZ,cinder,neutron也有对应的AZ概念),AZ一般依据地址，网络部署或电力配置划分，可以是一个独立的机房，或者一个独立供电的机架等，用户在创建instance的时候可以指定AZ，从而使instance创建在指定的AZ中，而host aggregate是一个面向管理员的概念，主要用来给nova-scheduler调度使用，比如根据某一属性（例如含有固态硬盘）划分一个host aggregate，把所有含有固态硬盘的host都放到该host aggregate中，nova-scheduler调度时指定相关属性就可以调度到对应host aggregate中的host。

![AZandHG](http://oeptotikb.bkt.clouddn.com/2018-08-08-az.png)
如上图，有两个地理隔离的Region，四个AZ，以及若干个根据不同属性区分的host aggregate。host aggregate 创建的时候可以指定AZ（如果AZ没有就会自动创建一个AZ），一个host可以属于多个host aggregate，但只能属于一个AZ。如下为一个示例：

```bash
nova aggregate­create storage­optimized storage­optimized-AZ  # 创建一个host aggregate
nova aggregate­set­metadata $aggregateID fast­storage=true # 设置 aggregate 的metadate
nova aggregate­add­host $aggregateID host­1 #添加host到aggregate
nova flavor­key $flavorID set fast­storage=true #添加flavor的 条件

```


### Workload segregation

负载这一块的策略是基于server-group来做的，相对比较简单。注意，这里的server-group不再是host的集合，而是instance的集合。比如，我要创建3个instance，因为这3个instance都比较吃内存，所以想要这3个instance在不同的host上，就可以这样做：

```bash
nova server-group-create --policy anti-affinity group-1 # 创建server-group，注意policy指定策略是不在同一节点
nova boot --image IMAGE_ID --flavor 1 --hint group=group-1 inst1 #创建instance1 
nova boot --image IMAGE_ID --flavor 1 --hint group=group-1 inst2 #创建instance2 
nova boot --image IMAGE_ID --flavor 1 --hint group=group-1 inst3 #创建instance3
```

在nova配置文件中指定scheduler策略，有一个针对server-group的filter叫做ServerGroupAntiAffinityFilter。除了anti-affinity，还有 affinity策略，就是尽量让同一server-group的instance在同一个host上。


## 参考文章


[理解openstack中region、cell、availability zone、host aggregate 概念](http://www.jianshu.com/p/613d34ad6d51)

[openstack运维实战系列(十二)之nova aggregate资源分组](http://happylab.blog.51cto.com/1730296/1739180)

[DIVIDE AND CONQUER:RESOURCE SEGREGATION IN THE OPENSTACK
CLOUD](https://www.openstack.org/assets/presentation-media/divideandconquer-2.pdf)

***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***