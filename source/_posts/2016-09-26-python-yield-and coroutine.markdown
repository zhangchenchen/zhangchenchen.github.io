---
layout: post
title:  "python系列--Python中的生成器以及协程的相关知识"
date:   2016-09-26 14:24:25
tags: python
---

## 目录结构：


[迭代器与生成器 ](#A)

[yield, send 关键字解析 ](#B)

[协程的一些知识](#C)






<a name="A"></a>

## 迭代器与生成器

在说生成器之前我们得先讨论一下迭代器的概念。

我们知道，对于一个可迭代的对象，我们可以通过 for * in * 的语句来进行迭代输出。这个可迭代的对象可以是一个list，string，文件等。其实我们深入这个可迭代的对象会发现，这些可迭代对象都实现了两个函数：__iter__() 和 next()。在我们调用list等这些可迭代对象的时候，需要把整个list数据全部读到内存里。这样就存在一个问题：list数据量小还可以，一旦变得特别大，内存就有可能被占满而导致运行缓慢甚至崩溃。这个时候我们想到，能不能只把我们需要的那个数据读入内存，调用next()的时候再把下一个数据读入内存呢。这就是生成器了。

一般来说，含有 yield 语句的函数就可叫做生成器。

```python

def g():
    yield 0
    yield 1
    yield 2
ge = g()
type(ge) #ge 的类型为 'generator'
dir(ge) #列出ge包含的函数我们发现有__iter__() 和 next()
ge.next()
ge.next()
ge.next()
```

生成器也是一种迭代器，但我们只可以读取它一次，因为它并非把所有数据都存入内存中，而是实时地生成数据。上述例子当我们再次读取ge.next()后就会报错。

此处再略提一下生成器推导式：
我们知道Python中有各种集合的推导式，如下：

 - 列表推导式：my_list = [ f(x) for x in sequence if cond(x) ]
 - 字典推导式：my_dict = { k(x): v(x) for x in sequence if cond(x) }
 - 集合推导式：my_set = { f(x) for x in sequence if cond(x) }

相应的，我们也可以用推导式来生成生成器，跟列表推导式类似，只需要将[]改为()：
my_generator = ( f(x) for x in sequence if cond(x) )




<a name="B"></a>

## yield, send 关键字解析

由上文可以看到，yield关键字是理解生成器的关键。那么yield什么意思呢？我们来详细解释一下。
yield 关键字有点类似我们平常写程序用到的return。但是程序运行到return的时候就会返回，运行完毕。而运行到yield的时候也会返回，但会保存上下文之后再返回，等到下次再次唤醒该程序的时候恢复上下文，继续从此运行，直到碰到下一次yield。

```python
 mygenerator = (x*x for x in range(3))
 for i in mygenerator :
    print(i) #输出0,1,4
```

首先我们创建了一个生成器mygenerator，注意，此时程序并不执行，只是创建了一个生成器。接着是一个for循环,fou循环其实就是执行了一个next()函数，此时进入生成器获取第一个i就是0乘以0，也就是0。返回并输出。再次循环，进入生成器，以此类推。这个例子可能不是特别直接，只是说明一下运行的逻辑顺序，下文中我们有一个更为直接的实例。

我们先看一下send()函数再将两个函数放在一个例子中说明一下。

在调用生成器时，除了next()方法，我们还可以用send()方法唤醒生成器，而且send(args),在唤醒生成器的同时会把参数 args传给指定的数据。如下例：

```python
def f():     #定义一个生成器函数
     while True:
         val = yield # val接受send()传过来的的参数并赋值
         yield val*10 

g = f() #新建一个生成器
g.next() #触发生成器，生成器执行到val=yield ，保存上下文退出，等待传值
g.send(1) # 触发生成器，生成器继续执行，赋值val = 1，执行到yield val*10，保存上下文，,返回10，退出。

g.next() #同上
g.send(10)

g.next()
g.send(0.5)

```

注意，我们在用send()之前必须保证生成器已经执行到yield,也就是说生成器已经被触发过一次，我们可以用send(none)来实现。其实，next()跟send(none)效果是一样的。

接下来我们看一个比较复杂的例子：

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-
# @Date    : 2016-09-22 16:40:36
# @Author  : Zhang Chen (pekingzcc@gmail.com)
# @Link    : zhangchenchen.github.io

import os

def countdown(n):
    print "Into genaration :counting down from", n
    while n >= 0:
        print "Genaration: ###loops from here###"
        newvalue = (yield n)
        print "Gerenation: newvalue is",newvalue
        # If a new value got sent in, reset n with it
        if newvalue is not None:
            n = newvalue
        else:
            n -= 1
if __name__=='__main__':
    c = countdown(5)
    print "Main function begin from here "
    for x in c:
         print "Main function: x is", x
         if x == 5:
            c.send(3)
```

这个例子看明白了，yield与send的用法就理解的差不多了。
我们首先看一下执行结果：

![python-yield](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/20160926python-yield.png)

接下来分析一下：我们从main 函数开始，先创建了一个生成器（此时并未执行），接着输出main函数开始的语句。接着进入for循环，注意，遇到for循环就相当于执行了一次next(),所以进入生成器输出“Into genaration :counting down from 5”，继续运行，进入生成器的for循环，输出“Genaration: ###loops from here###”，继续往下，碰到yield n ，保存上下文，退出并返回n(此时是5)到main函数，主函数输出“Main function: x is 5”，进入条件语句，c.send(5)触发生成器，再次进入生成器，赋值newvalue为3，接着输出“Gerenation: newvalue is 3” ，赋值n = newvalue =3 ,继续循环输出“Genaration: ###loops from here###”，碰到yield n ,返回main函数，注意，此时main函数又进入到for循环，所以再次进入生成器，但是没有send()数据传过来，也可以理解为send(none),所以输出“Gerenation: newvalue is none”,接着执行n-1 ,n变为2。继续循环，输出“Genaration: ###loops from here###”。。。。。



<a name="C"></a>

## 协程的一些知识

在了解协程之前，我们需要先从进程，线程说起。我们知道，进程的出现是为了并发，在一台机器上同时运行多个程序（当然，内部实现可能是多CPU并行，也有可能是单CPU时间分片，但在外部看来就是多个程序一起运行），进程的切换需要陷入内核，由OS来进行切换。一切换进程得反复进入内核，置换掉一大堆状态，这样，进程数一高，就会吃掉很多的系统资源。为了解决这个问题，就出现了线程的概念。

一个进程里可以有多个线程，这样就能处理多个逻辑，当某个线程阻塞的时候，可以切换线程到另一个线程。因为线程是共享附属进程的资源的，它们件的切换相对要比进程间的切换消耗的资源要少很多。但之后，问题又来了，操作系统为了程序运行的高效性每个线程都有自己缓存Cache等等数据，操作系统还需要做这些数据的恢复操作，所以线程多了之后，它们之间的切换也非常耗性能。

协程的概念就来了，既然线程切换费资源，那我们干脆自己做逻辑流的切换，不用交给OS处理。注意，这里经常提到的是切换，具体到应用场景就是IO密集型场景。在cpu密集型的场景中协程的意义就没有那么大了。在IO处理时，我们有同步，异步两种处理方式，关于IO模型，可以[check this](http://www.jianshu.com/p/55eb83d60ab1) 。在异步处理IO时，我们需要写回调函数来实现异步，这种写法是比较反人类的，可读性比较差。协程可以很好解决这个问题。比如 把一个IO操作 写成一个协程。当触发IO操作的时候就自动让出CPU给其他协程。协程的切换很轻，消耗资源少。协程通过这种对异步IO的封装既保留了性能也保证了代码的 容易编写和可读性。

其实协程不是一个新生的事物，它在很早之前就出现了，只不过最近因为在一些动态语言的世界里大放异彩。对其历史感兴趣的[check this](http://blog.youxu.info/2014/12/04/coroutine/)。

再简单说一下Python中的协程，其实Python中的协程跟yield关键字是分不开的。只要含有yield的函数都可以认为是一个协程，利用协程实现的生产者消费者如下：

```python
import random
 
def get_data():
    """返回0到9之间的3个随机数，模拟异步操作"""
    return random.sample(range(10), 3)
 
def consume():
    """显示每次传入的整数列表的动态平均值"""
    running_sum = 0
    data_items_seen = 0
    
    while True:
        print('Waiting to consume')
        data = yield
        data_items_seen += len(data)
        running_sum += sum(data)
        print('Consumed, the running average is {}'.format(running_sum / float(data_items_seen)))
 
def produce(consumer):
    """产生序列集合，传递给消费函数（consumer）"""
    while True:
        data = get_data()
        print('Produced {}'.format(data))
        consumer.send(data)
        yield
 
if __name__ == '__main__':
    consumer = consume()
    consumer.send(None) 
    producer = produce(consumer)
 
    for _ in range(10):
        print('Producing...')
        next(producer)
```

利用协程的Python库比较常见的是Greenlet库，它是以C扩展模块形式接入Python的轻量级协程，将一些原本同步运行的网络库以mockey_patch的方式进行了重写。Greenlets全部运行在主程序操作系统进程的内部，它们被协作式地调度。



## 参考文章

[A Curious Course on Coroutines and Concurrency](http://www.dabeaz.com/coroutines/Coroutines.pdf)

[提高你的Python: 解释‘yield’和‘Generators（生成器）](https://www.oschina.net/translate/improve-your-python-yield-and-generators-explained)

[(译)Python关键字yield的解释(stackoverflow)](http://pyzh.readthedocs.io/en/latest/the-python-yield-keyword-explained.html)

[Python 中的进程、线程、协程、同步、异步、回调](https://segmentfault.com/a/1190000001813992)

[什么是Python中的生成器推导式？](http://codingpy.com/article/what-is-generator-comprehension/)

[生成器](http://wiki.jikexueyuan.com/project/start-learning-python/215.html)

[聊聊 Python 中生成器和协程那点事儿](http://manjusaka.itscoder.com/2016/09/11/something-about-yield-in-python/)

[GENERATOR.SEND() WITH YIELD](http://www.bogotobogo.com/python/python_function_with_generator_send_method_yield_keyword_iterator_next.php)

[了解协程（Coroutine）](http://blog.kazaff.me/2016/05/29/%E4%BA%86%E8%A7%A3%E5%8D%8F%E7%A8%8B(coroutine)/)

[协程的好处是什么？](https://www.zhihu.com/question/20511233)

[为什么觉得协程是趋势？](https://www.zhihu.com/question/32218874/answer/55469714)

[编程珠玑番外篇-Q 协程的历史，现在和未来](http://blog.youxu.info/2014/12/04/coroutine/)

[利用python yielding创建协程将异步编程同步化](http://www.jackyshen.com/2015/05/21/async-operations-in-form-of-sync-programming-with-python-yielding/)

***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***
