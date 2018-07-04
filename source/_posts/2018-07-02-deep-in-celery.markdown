---
layout: post
title:  "Inf-- celery 源码大致解析"
date:   2018-07-03 14:57:10
tags: 
  - celery
  - Inf
---


## 概述

最近一直在用celery 开发一个自动化运维的项目，celery 的文档看完之后，虽然使用起来没什么问题了，不过感觉celery 对我还是一个黑盒，就想在源码级别大致了解一下celery，故有了这篇文章。

tips：本来是想把项目clone 下来花个半天的时间大致过一遍，结果发现celery 比我想象的要复杂很多，时间原因，放弃此法，转而去找现成的源码分析，拾人牙慧，难免纰漏。


## celery 与 kombu

python 基于AMQP的库，比较常用的有两个，pika 和 kombu。kombu 相对pika 是更高层面的抽象，pika 只支持AMQP 0.9.1协议，而kombu 抽象了中间的broker，可以支持多种broker（redis,zookeeper,mongodb等）。而且相对pika 提供了很多特性：重连策略，连接池，failover 策略等等。这些策略都是一些常用且比较重要的特性，如果用pika 的话需要自己去造轮子。
kombo 更像是celery 的定制库，在celery中大量使用了kombu中的概念，kombu的更底层是调用的librabbitmq 库或py-amqp库来实现AMQP 0.9.1 ，在这个层面上，pika更接近py-amqp库。这里再提一嘴，open stack 项目在kombu的基础上又针对性的封装了一层，就是著名的oslo.messaging公共库了。
所以，如果想研究celery，那么最好先好好研究下kombo。


## 从使用探究内部原理

这里就不在源码层面一层层的分析了，可以参考下面参考文章的内容，有此类的分析。这里主要是从使用层面大致说下在使用的过程中，celery 内部做了哪些事情。
我们使用celery的话，可能最主要的就是两个事情，一个是创建要执行的任务，然后用app.task 进行装饰，第二个是启用celery worker 进程。这里就从这两个地方入手，大致说下celery 的实现机制。


### celery worker 

首先了解一个blueprint 的概念，跟flask里面的概念类似，都是一个分组的概念。celery worker 启动的时候会有两个blueprint，一个是worker，一个是consumer，这两个blueprint都包含一些bootstep,bootstep 可以认为是启动的组件,bootstep 之间是有依赖关系的，也就是说，他们的启动是有顺序的，这些bootstep 很多引用了kombu中的概念。 

如下是worker 的bootstep:
![celery-blueprint](http://7xrnwq.com1.z0.glb.clouddn.com/celery-blueprint.png)
简要介绍下各个bootstep 的作用：

Worker bootstep：
- Timer：用于执行定时任务的 Timer，和 Consumer 那里的 timer 不同
- Hub：Eventloop 的封装对象（复用的 Kombu ）
- Pool：构造各种执行池（线程/进程/协程）的
- Autoscaler：用于自动增长或者 pool 中工作单元
- StateDB：持久化 worker 重启区间的数据（只是重启）
- Autoreloader：用于自动加载修改过的代码
- Beat：创建 Beat 进程，不过是以子进程的形式运行（不同于命令行中以 beat 参数运行）

Consumer bootstep：
- Connection：管理和 broker 的 Connection 连接
- Events：用于发送监控事件
- Agent：cell actor
- Mingle：不同 worker 之间同步状态用的
- Tasks：启动消息 Consumer
- Gossip：消费来自其他 worker 的事件
- Heart：发送心跳事件（consumer 的心跳）
- Control：远程命令管理服务

各个bootstep 的依赖关系：
![bootstep](http://7xrnwq.com1.z0.glb.clouddn.com/2018-07-03-bootstep.png)

这些bootstep 的具体行为这里不做解释了，感兴趣的可以去看参考文章，celery这里还留了一个接口，可以让我们自己 custom bootstep，自定义一个bootstep 类，然后hook 一些worker 在不同阶段会执行的自定义动作。参考[Extensions and Bootsteps](http://docs.celeryproject.org/en/latest/userguide/extending.html#blueprints)


### celery task

在定义task 的时候，我们经常使用这样一段类似代码：

```
@app.task
def add(x,y):
    return x+y 
```

执行代码看下这个add其实是一个task 对象，看下他的具体调用apply_async()函数，这里会分成两个分支，一个如果是指定同步，执行走同步的逻辑，异步会走异步的逻辑。中间都会经过一些判断逻辑（信号处理，失败处理，依赖处理等）然后会进入具体的执行逻辑，也就是消息体的创建以及发送，看下最终的消息体format：

```
properties = {
    'correlation_id': uuid task_id,
    'content_type': string mimetype,
    'content_encoding': string encoding,

    # optional
    'reply_to': string queue_or_url,
}
headers = {
    'lang': string 'py'
    'task': string task,
    'id': uuid task_id,
    'root_id': uuid root_id,
    'parent_id': uuid parent_id,
    'group': uuid group_id,

    # optional
    'meth': string method_name,
    'shadow': string alias_name,
    'eta': iso8601 ETA,
    'expires': iso8601 expires,
    'retries': int retries,
    'timelimit': (soft, hard),
    'argsrepr': str repr(args),
    'kwargsrepr': str repr(kwargs),
    'origin': str nodename,
}

body = (
    object[] args,
    Mapping kwargs,
    Mapping embed {
        'callbacks': Signature[] callbacks,
        'errbacks': Signature[] errbacks,
        'chain': Signature[] chain,
        'chord': Signature chord_callback,
    }
)
```


其实还有很多比如 events 的实现，task state & result 等都是值得探讨研究的，止笔于此，后续有需求再看。

## 参考文章

[Kombu 4.2.1 documentation](http://docs.celeryproject.org/projects/kombu/en/latest/introduction.html)

[kombu和消息队列总结](http://gtcsq.readthedocs.io/en/latest/openstack/kombu.html)

[Celery 源码解析](https://liuliqiang.info/post/celery-source-analysis-worker-execute-engine)

[Celery源码分析（一）-从命令执行到生成Worker](https://blog.csdn.net/happyAnger6/article/details/53869262)

[pika vs kombu](https://stackoverflow.com/questions/48524536/can-anyone-please-tell-me-what-are-the-differences-between-pika-and-kombu-messag)


***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***