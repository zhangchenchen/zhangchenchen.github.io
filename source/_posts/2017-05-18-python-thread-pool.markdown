---
layout: post
title:  "python-- python threadpool 的前世今生"
date:   2017-05-18 10:21:32
tags: 
  - python
  - threadpool
---




## 引出

首先需要了解的是threadpool 的用途，他更适合于用到一些大量的短任务合集，而非一些时间长的任务，换句话说，适合大量的CPU密集型短任务，那些消耗时间较长的IO密集型长任务适合用[协程](https://zhangchenchen.github.io/2016/09/26/python-yield-and%20coroutine/)去解决。

目前，python 标准库（特指python2.X）中的threadpool模块是在 multiprocessing.pool.threadpool，或者multiprocessing.dummy.ThreadPool（dummy模块是针对threading 多线程的进一步封装）。该模块有个缺点就是在所有线程执行完之前无法强制退出。实现原理大同小异：实例化pool的时候会创建指定数目的线程，把task 传给一个task-queue，线程会读取task-queue 的task，没有就阻塞，读取到后就执行，并将结果交给一个result-queue。

除了标准库中的threadpool，还有一些使用比较多的threadpool，以下展开。

## pip 中的 ThreadPool

安装简单：pip install threadpool 
使用如下：

```python
pool = ThreadPool(poolsize)   # 定义线程池，指定线程数量
requests = makeRequests(some_callable, list_of_args, callback) # 调用makeRequests创建了要开启多线程的函数，以及函数相关参数和回调函数  
[pool.putRequest(req) for req in requests]  # 所有要运行多线程的请求扔进线程池
pool.wait()  # 等待所有线程完成后退出

```

原理类似，源码解读可以参考[python——有一种线程池叫做自己写的线程池](http://www.cnblogs.com/Eva-J/p/5106564.html) ,该博客还给出了对其的一些优化。



## 自己定制 threadpool

根据需要的功能定制适合自己的threadpool 也是一种常见的手段，常用的功能比如：是否需要返回线程执行后的返回值，线程执行完之后销毁还是阻塞等等。以下为自己经常用的的一个比较简洁的threadpool，感谢@kaito-kidd提供，[源码](https://github.com/kaito-kidd/thread_pool/blob/master/pool.py):



```python
# coding: utf8

"""
线程池，用于高效执行某些任务。
"""

import Queue
import threading


class Task(threading.Thread):

    """ 任务  """

    def __init__(self, num, input_queue, output_queue, error_queue):
        super(Task, self).__init__()
        self.thread_name = "thread-%s" % num
        self.input_queue = input_queue
        self.output_queue = output_queue
        self.error_queue = error_queue
        self.deamon = True

    def run(self):
        """run
        """
        while 1:
            try:
                func, args = self.input_queue.get(block=False)
            except Queue.Empty:
                print "%s finished!" % self.thread_name
                break
            try:
                result = func(*args)
            except Exception as exc:
                self.error_queue.put((func.func_name, args, str(exc)))
            else:
                self.output_queue.put(result)


class Pool(object):

    """ 线程池 """

    def __init__(self, size):
        self.input_queue = Queue.Queue()
        self.output_queue = Queue.Queue()
        self.error_queue = Queue.Queue()
        self.tasks = [
            Task(i, self.input_queue, self.output_queue,
                 self.error_queue) for i in range(size)
        ]

    def add_task(self, func, args):
        """添加单个任务
        """
        if not isinstance(args, tuple):
            raise TypeError("args must be tuple type!")
        self.input_queue.put((func, args))

    def add_tasks(self, tasks):
        """批量添加任务
        """
        if not isinstance(tasks, list):
            raise TypeError("tasks must be list type!")
        for func, args in tasks:
            self.add_task(func, args)

    def get_results(self):
        """获取执行结果集
        """
        while not self.output_queue.empty():
            print "Result: ", self.output_queue.get()

    def get_errors(self):
        """获取执行失败的结果集
        """
        while not self.error_queue.empty():
            func, args, error_info = self.error_queue.get()
            print "Error: func: %s, args : %s, error_info : %s" \
                % (func.func_name, args, error_info)

    def run(self):
        """执行
        """
        for task in self.tasks:
            task.start()
        for task in self.tasks:
            task.join()


def test(i):
    """test """
    result = i * 10
    return result


def main():
    """ main """
    pool = Pool(size=5)
    pool.add_tasks([(test, (i,)) for i in range(100)])
    pool.run()


if __name__ == "__main__":
    main()


```




## 参考文章

[Python Thread Pool](https://www.metachris.com/2016/04/python-threadpool/)

[理解python的multiprocessing.pool threadpool多线程](http://xiaorui.cc/2015/11/03/%E7%90%86%E8%A7%A3python%E7%9A%84multiprocessing-pool-threadpool%E5%A4%9A%E7%BA%BF%E7%A8%8B/)

[thread_pools.py](https://gist.github.com/BeginMan/0afc01a5a01470372a0e3399322d233d)

[Cinder磁盘备份原理与实践](http://www.cnblogs.com/xiaozi/p/6182990.html)

[python线程池（threadpool）模块使用笔记](http://blog.csdn.net/wytdahu/article/details/45246095)

[thread_pool](https://github.com/kaito-kidd/thread_pool/blob/master/pool.py)


***END***
