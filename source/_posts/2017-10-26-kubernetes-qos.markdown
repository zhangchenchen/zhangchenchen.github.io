---
layout: post
title:  "kubernetes-- kubernetes scheduler设计以及resource QOS"
date:   2017-10-26 15:03:10
tags: 
  - kubernetes
---


## kubernetes scheduler 设计

kubernetes最主要的功能是容器编排，而容器编排中第一个想到的便是新建一个容器时，需要将该容器调度到哪一个节点上，其实跟openstack中的nova-scheduler很像。
看下我们设计sheduler时需要满足的需求：
- 从用户角度，需要满足指定节点，与指定pod亲和/反亲和（例如同为cpu密集型的服务不要放在同一节点），服务分散
- 从集群的角度，需要满足资源平衡，利用率高等

看下kubernetes中大致的调度过程，图片来自[Kubernetes中的资源管理 ](http://oj6ydypm2.bkt.clouddn.com/Kubernetes%E4%B8%AD%E7%9A%84%E8%B5%84%E6%BA%90%E7%AE%A1%E7%90%86_%E4%B8%81%E6%B5%B7%E6%B4%8B.pdf)：

![k8s-scheduler](http://7xrnwq.com1.z0.glb.clouddn.com/20171026k8s-scheduler.jpg)

再看下调度器的具体调度算法，大致分了两类，predicate和prioritizer：
- predicate:一系列过滤函数将不满足的node排除
- prioritizer: 对通过的节点按照一系列的算法进行优先级排序，最后选择优先级最高的。

![kube-scheduler-algori](http://7xrnwq.com1.z0.glb.clouddn.com/20171026-kuber-scheduler-argo.jpg)

简单介绍下常用的predicate 的几个算法：

- PodFitsResources：节点上剩余的资源是否大于 pod 请求的资源
- PodFitsHost：如果 pod 指定了 NodeName，检查节点名称是否和 NodeName 匹配
- PodFitsHostPorts：节点上已经使用的 port 是否和 pod 申请的 port 冲突
- PodSelectorMatches：过滤掉和 pod 指定的 label 不匹配的节点
- NoDiskConflict：已经 mount 的 volume 和 pod 指定的 volume 不冲突，除非它们都是只读

prioritizer的常用算法：

- LeastRequestedPriority：将Pod部署到剩余CPU和内存最多的Node上，这里的request就是kubernetes的QOS中定义的request。
- BalancedResourceAllocation：节点上 CPU 和 Memory 使用率越接近，权重越高。这个应该和上面的一起使用，不应该单独使用
- ImageLocalityPriority：倾向于已经有要使用镜像的节点，镜像总大小值越大，权重越高
- NodeLabelPriority ：检查Node是否存在Pod需要的Label，有则10分，没有则0分
- Spreading：同一个Service下的Pod尽可能的分散在集群中。Node上运行的通Service
下的Pod数目越少，分数越高。
- Anti-Affinity：与Spreading类似，但是分散在拥有特定Label的所有Node中，包括Node Affinity/Anti-Affinity和Pod Affinity/Anti-Affinity。

此外，1.6之后还有Taints and Tolerations等scheduler策略，当然，也可以定制，然后创建pod时指定schedulername,参考[Advanced Scheduling in Kubernetes](http://blog.kubernetes.io/2017/03/advanced-scheduling-in-kubernetes.html)。



## kubernetes resource Quality of Service(QOS)


kubernetes中的QOS实现是通过request和limit两个概念,主要包括两个具体的指标：CPU和内存。request可以理解为pod的下限，即最少分配给该pod的资源，limit则相应的是分配资源的上限。
对于每一个资源，container可以指定具体的资源需求（requests）和限制（limits），requests申请范围是0到node节点的最大配置，而limits申请范围是requests到无限，即0 <= requests <=Node Allocatable, requests <= limits <= Infinity。
对于CPU，如果pod中服务使用CPU超过设置的limits，pod不会被kill掉但会被限制。如果没有设置limits，pod可以使用全部空闲的cpu资源。
对于内存，当一个pod使用内存超过了设置的limits，pod中container的进程会被kernel因OOM kill掉。当container因为OOM被kill掉时，系统倾向于在其原所在的机器上重启该container或本机或其他重新创建一个pod。
在Kubernetes中，pod的QoS级别分为以下三种：Guaranteed, Burstable与 Best-Effort。

### Guaranteed 

满足条件：pod中所有容器都必须统一设置limits，并且设置参数都一致，如果有一个容器要设置requests，那么所有容器都要设置，并设置参数同limits一致。如果一个容器只指明limit而未设定request，则request的值等于limit值。

Guaranteed举例1：容器只指明了limits而未指明requests。

```json
containers:
name: foo
resources:
  limits:
    cpu: 10m
    memory: 1Gi
name: bar
resources:
  limits:
    cpu: 100m
    memory: 100Mi
```
Guaranteed举例2：requests与limit均指定且值相等。
```json
containers:
name: foo
resources:
  limits:
    cpu: 10m
    memory: 1Gi
  requests:
    cpu: 10m
    memory: 1Gi

name: bar
resources:
  limits:
    cpu: 100m
    memory: 100Mi
  requests:
    cpu: 100m
    memory: 100Mi
```

### Best-Effort

满足条件：Pod中所有容器的所有Resource的request和limit都没有赋值。

示例：
```json
containers:
    name: foo
        resources:
    name: bar
        resources:
```


### Burstable

满足条件：pod中只要有一个容器的requests和limits的设置不相同，满足“0<request<limit<∞” 。

示例,Container bar没有指定resources：

```json
containers:
    name: foo
        resources:
            limits:
                cpu: 10m
                memory: 1Gi
            requests:
                cpu: 10m
                memory: 1Gi

    name: bar
```


3种QoS优先级从有低到高（从左向右）：
Best-Effort pods -> Burstable pods -> Guaranteed pods

也就是说，QoS pods被kill掉场景与顺序如下： 
- Best-Effort 类型的pods：系统用完了全部内存时，该类型pods会最先被kill掉。
- Burstable类型pods：系统用完了全部内存，且没有Best-Effort container可以被kill时，该类型pods会被kill掉。
- Guaranteed pods：系统用完了全部内存、且没有Burstable与Best-Effort container可以被kill，该类型的pods会被kill掉。
注：如果pod进程因使用超过预先设定的limites而非Node资源紧张情况，系统倾向于在其原所在的机器上重启该container或本机或其他重新创建一个pod。



## 参考文章


[Kubernetes中的资源管理 ](http://oj6ydypm2.bkt.clouddn.com/Kubernetes%E4%B8%AD%E7%9A%84%E8%B5%84%E6%BA%90%E7%AE%A1%E7%90%86_%E4%B8%81%E6%B5%B7%E6%B4%8B.pdf)

[kubernetes 简介：调度器和调度算法](http://cizixs.com/2017/03/10/kubernetes-intro-scheduler)

[How does the Kubernetes scheduler work](https://jvns.ca/blog/2017/07/27/how-does-the-kubernetes-scheduler-work/)


[Kubernetes计算资源管理--requests和limits](http://blog.csdn.net/yan234280533/article/details/62965993)

[Kubernetes之服务质量保证（QoS）](http://dockone.io/article/2592)

[Configure Quality of Service for Pods](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/)

[resource-qos.md](https://github.com/eBay/Kubernetes/blob/master/docs/proposals/resource-qos.md)



***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***