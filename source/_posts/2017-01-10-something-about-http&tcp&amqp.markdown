---
layout: post
title:  "web系列--关于http，tcp,amqp的一些思考"
date:   2017-01-10 10:17:25
tags: web
---

## 目录结构：

[为什么TCP是可以保持长连接的，而基于TCP的HTTP却不能 ](#A)

[AMQP VS HTTP](#B)


<a name="A"></a>

## 为什么TCP是可以保持长连接的，而基于TCP的HTTP却不能 

- 可能这个问题本身表述就不是很清楚，但我也想不出其他合适的词汇了。直接上例子吧：我们在做web应用的时候有时会碰到需要实时推送的情况，但是HTTP协议本身是不支持长时间的连接的，因为HTTP本身是request/response型的协议，可以理解为半双工通讯。而TCP 是支持长连接的，是通过heartbeat来进行连接的保持。这样，就引出我上面提到的问题了。
- 在解答这个问题之前，还有几点是要说明的：http1.1中其实也有长连接的概念，具体到header参数就是keep-alive，但是这个keep-alive是一个数值，比如20s,就是这个http连接对应的tcp连接会保持20S,这个时间段过来的http连接会复用上次的tcp连接。如果没有的话，那么会开启一个新的tcp连接。而tcp协议里面的keep-alive就是heartbeat发送的包内容，表示“我还活着”。
- 接下来回答一下上面那个问题：其实很简单，我们混淆了“基于TCP协议”这个概念，TCP本身是一个传输层的协议，就是用来作数据传输的，也决定了它必然是全双工，可以保持连接的特性。再来看下http协议，一种request/response型的协议，为什么要做成request/response这种类型呢，因为server端要服务的client太多了，如果每一个连接都保持而不断开，那么服务器的IO很快就会被堵死了。所以http基于tcp,并非就是全部利用的tcp的特性，在http连接建立之前，的确是需要tcp三次握手建立tcp连接，然后利用这个tcp连接传输http包数据，服务端收到请求的数据后，发送对应的response,之后便关闭了这个tcp连接（如果没有keep-alive）。
- 再回到刚才的例子：其实现在解决这种实时推送的方案还是很丰富的，比较主流的应该是利用websocket协议来实现。当然，其他的比如：ajax实现的轮询等。就不细讲了，感兴趣就自行Google吧。
    


<a name="B"></a>

## AMQP VS HTTP

其实这个问题应该这样描述：在做一个SOA服务时，用HTTP好还是AMQP好？

该问题的提出主要是因为openstack的内部通信中主要包含rest和Rabbimq两种通信方式，所以不禁对于两种方式的优劣提出了疑问，经过一番思考以及搜索资料后，找到了相对满意的答案：

主要参考[When designing Service Oriented Architectures, which one do you use - HTTP or AMQP?](https://hashnode.com/post/when-designing-service-oriented-architectures-which-one-do-you-use-http-or-amqp-ciibz8flr01drj3xt1ga99f3w)

 - AMQP is damn awesome! 确实，amqp更快，可定制性更强，可以实现异步传输，相对于http来说，作为SOA的信息传输协议真是再好不过了。简单列举下AMQP碾压HTTP 的优点，此处以rabbitmq为例：
    1. rabbitmq可以提供面向连接，可靠的信息传递。
    2. rabbitmq提供过载保护：比如设置queue的timeout,限制 message rate,甚至转移message到别的queue.
    3. http需要有一个service discovery 机制，rabbitmq 不用。
    4. rabbitmq 的loadbalance 使用非常简单
    5. rabbitmq 的后端如果做了热备，如果一个后端节点挂掉，会自动切换到另一节点。
 - 再说下http的优点，毕竟年岁要长一些，SOA的框架非常成熟，学习的曲线也不高，lz之前用过阿里出品的dubbo框架，简单易用。更重要的是，http作为一种成熟的协议标准，想要满足任何需求基本都会有对应的中间件，而且scale up 的成本低。
 - 综上所述，不难看出，在真正用到SOA时，还是需要取决于实际情况来取舍。如果只是一个内部的系统，交互的模块相对紧密的话推荐用AMQP。如果是一个大型的web系统，与外部交互比较多，而且是多部门协作完成多个模块的开发，那么模块件的交互还是使用标准的http吧。
 - 对应到openstack也可以看出一二：模块之间交互都是使用http/rest,比如nova与keystone的认证交互，因为每个独立的模块都是一个项目，都有独立的开发团队。而在模块内部，比如，Nova内部，nova-api与nova-scheduler,nova-scheduler与nova-compute 都是使用amqp来实现。






## 参考文章


[The Problem We Solve](http://www.amqp.org/product/solve)

[用两条 HTTP 请求就可以实现双向实时通讯吧？为什么还需要 WebSocket](https://www.v2ex.com/t/170762)

[微服务与SOA之间差了一个ESB](http://blog.csdn.net/jdk2006/article/details/51695416)

[When designing Service Oriented Architectures, which one do you use - HTTP or AMQP?](https://hashnode.com/post/when-designing-service-oriented-architectures-which-one-do-you-use-http-or-amqp-ciibz8flr01drj3xt1ga99f3w)


[关于 RabbitMQ 的实时消息推送](https://www.ibm.com/developerworks/cn/opensource/os-cn-rabbit-mq/)



***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***
