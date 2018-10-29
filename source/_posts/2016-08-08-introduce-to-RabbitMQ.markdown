---
layout: post
title:  "opensatck 系列--RabbitMQ 快速入门"
date:   2016-08-08 16:06:25
tags: openstack
---


## 简介

RabbitMQ 是一个消息队列框架，一个message broker，它的主要工作就是接收并转发消息。类似生产者/消费者模型，它的三个主要角色也是生产者（产生消息的角色），消费者（处理消息的角色），消息队列（缓存消息），接下来介绍下RabbitMQ的几个经典的应用模式。

## 准备工作

[RabbitMQ 安装](https://www.rabbitmq.com/download.html)

[封装了AMQP的python第三方库Pika](https://pika.readthedocs.io/en/0.10.0/intro.html)

## 发布/订阅 模式

所谓发布/订阅是指：producer 发布消息，订阅的consumer都会接收到此消息，类似广播。
示例给出一个简单的日志系统，产生的日志信息会广播到所有的consumer。

producer 代码如下，解释在注释中：

```python
import pika  # 引入pika包
import sys

connection = pika.BlockingConnection(pika.ConnectionParameters(
        host='localhost'))     #指定主机host
channel = connection.channel() #创建channel

channel.exchange_declare(exchange='logs',
                         type='fanout') #声明exchange，以及exchange的type，解释见下文
#channel.queue_declare(queue='hello') 
#创建一个名为hello的queue，因为此处是用的rabbitmq自动创建模式，所以注释掉
message = ' '.join(sys.argv[1:]) or "info: Hello World!" #要发送的消息
channel.basic_publish(exchange='logs',
                      routing_key='',
                      body=message) #绑定exchange，消息等
print(" [x] Sent %r" % message)
connection.close() # 关闭连接，关闭前会确定queue中是否还有消息，没有则关闭
```

Exchanges 解释：
你可以把exchange看作是producer与queue之间的一个分发器，它会按照指定的策略把消息分发到指定queue中。
![rabbitmq-exchange](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/rabbitmq_exchange.jpg)

exchange 的类型有四种，

 - direct 与routing_key 结合使用，指定发送到某个queue
 - topic 按照某种主题模式，指定发送到满足该模式的多个queue
 - headers RPC模式
 - fanout 广播模式

consumer 代码如下：

 ```python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters(
        host='localhost'))
channel = connection.channel()

channel.exchange_declare(exchange='logs',
                         type='fanout')

result = channel.queue_declare(exclusive=True) 
#没有指定名字所以创建一个临时queue,而且consume断开后该queue会被删除
queue_name = result.method.queue

channel.queue_bind(exchange='logs',
                   queue=queue_name) # 绑定exchange与queue

print(' [*] Waiting for logs. To exit press CTRL+C')

def callback(ch, method, properties, body):  # 被调用的处理逻辑函数
    print(" [x] %r" % body)

channel.basic_consume(callback,
                      queue=queue_name,
                      no_ack=True) #指定callback函数，queue,no_ack解释见下文

channel.start_consuming() 
 ```

ACK(消息确认机制)
在rabbitmq运行过程中，如果某个consumer挂掉，那么分配到该consumer的所有其他任务也就无法完成。为了防止这种情况，默认ACK机制开启，consumer完成某个任务后会返回queue一个ack,表示当前任务已完成，可以分配下一个任务。如果指定no_ack=True，那么就会关闭ack机制。


## 其余message分发机制

[routing 分发](https://www.rabbitmq.com/tutorials/tutorial-four-python.html)

[topic 分发](https://www.rabbitmq.com/tutorials/tutorial-five-python.html)

[RPC调用](https://www.rabbitmq.com/tutorials/tutorial-six-python.html)


## 参考文章

[RabbitMQ doc](https://www.rabbitmq.com/getstarted.html)


