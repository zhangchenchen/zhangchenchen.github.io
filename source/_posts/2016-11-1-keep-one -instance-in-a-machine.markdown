---
layout: post
title:  "linux shell系列-- 如何在系统中保证只有一个实例"
date:   2016-11-1 13:06:25
tags: linux
---

## 目录结构：

[如何在系统中保证只有一个实例 ](#A)

[写 pid 方式 ](#B)

[文件锁的方式](#C)

[独占端口的方式](#D)

[ dbus API](#E)

[写成service并交由第三方托管](#F)



<a name="A"></a>

## 如何在系统中保证只有一个实例

从这发现的这个问题，[如何确保系统中某个程序只存在一个实例](https://www.v2ex.com/t/316723#reply21)

从评论区中汇总几种可行的方案并实验，记录如下，。



<a name="B"></a>

## 写 pid 方式 

这种方式应该是最直观，最容易想到的，类似于算法中的暴力破解法。原理就是在程序执行之前将PID写入某个约定好的文件，每次程序启动前通过检测该PID对应的程序是否存在来判断。它的伪代码如下：

 > Invariant:
    File xxxxx will exist if and only if the program is running, and the
    contents of the file will contain the PID of that program.

    On startup:
        If file xxxxx exists:
            If there is a process with the PID contained in the file:
                Assume there is some instance of the program, and exit
            Else:
                Assume that the program terminated abnormally, and
                overwrite file xxxx with the PID of this program
        Else:
            Create file xxxx, and save the current PID to that file.

    On termination (typically registered via atexit):
        Delete file xxxxx

这种方式还要注意假如两个程序同时读取文件的时候这种情况，可以再加一个文件锁（其实就是下面的方式了）。

<a name="C"></a>

## 文件锁的方式

对文件锁的概念其实一直比较陌生，但其实与我们还是挺接近的。我们数据库中经常出现锁表或者锁记录的情况，就是利用文件锁实现的。关于文件锁的理论知识可以参考如下一篇文章：[Linux 2.6 中的文件锁](https://www.ibm.com/developerworks/cn/linux/l-cn-filelock/)

一个简单的demo如下：

```python
#!/usr/bin/env python
import fcntl
import os, time

FILE = "flag-file.txt"

if not os.path.exists(FILE):
    file = open(FILE,"w")
    file.write(""+os.getpid())
    file.close()

file = open(FILE,"r+")
try:
    fcntl.flock(file.fileno(),fcntl.LOCK_EX | fcntl.LOCK_NB)
    print "acquire lock, i am running"
    time.sleep(10)    
except IOError:
    print "another programme is running"

```

效果如下：
![demo](http://7xrnwq.com1.z0.glb.clouddn.com/2016-11-01fcntl.png)
 
使用这种方式有个地方要注意：你不知道什么时候别人可能把你作为flag的文件给删除了。当然可以通过逻辑判断的方式来解决，但当程序比较复杂的时候可能还会出现意料外的问题。所以我们可以用端口独占的方式来避免这种情况。

<a name="D"></a>

## 独占端口的方式

同上，通过独占端口的方式来达到互斥的效果，demo如下：

```python
#!/usr/bin/env python
import socket
import os, time

PORT = 888

def check_port(ip,port):
    s = socket.socket(socket.AF_INET,socket.SOCK_STREAM)
    try:
        s.bind((ip,int(port)))
        print "port is open,i'm running"
        time.sleep(10)
    except:
        print "another programme is running"

if __name__ == '__main__':
    check_port("127.0.0.1",PORT)

```


<a name="E"></a>

## dbus API

首先简单介绍下什么是 dbus,直接引用wiki：
 >D-Bus是一个进程间通信及远程过程调用机制，可以让多个不同的计算机程序（即进程）在同一台电脑上同时进行通信[4]。D-Bus作为freedesktop.org项目的一部分，其设计目的是使Linux桌面环境（如GNOME与KDE等）提供的服务标准化。

 dbus 的Python库主要有几下几个

 1,GDbus and QtDbus are wrappers over the C/C++ APIs of GLib and Qt

 2,pydbus is a modern, pythonic API with the goal of being as invisible a layer between the exported API and its user as possible
 
 3,dbus-python is a legacy API, built with a deprecated dbus-glib library, and involving a lot of type-guessing (despite "explicit is better than implicit" and "resist the temptation to guess").
 
 4,txdbus is a native Python implementation of the D-Bus protocol for the Twisted networking framework.

demo代码如下，因为对dbus这一块不熟，好像有个地方参数不对，后续再调吧

```python
#!/usr/bin/env python
import dbus
import dbus.service
import dbus.mainloop.glib
import os, time


def check_dbus(bus_name):
    dbus.mainloop.glib.DBusGMainLoop(set_as_default = True)
    #bus = dbus.SessionBus()
    # s = socket.socket(socket.AF_INET,socket.SOCK_STREAM)
    try:
        my_bus_name = dbus.service.BusName(bus_name,
                    bus, allow_replacement = False, replace_existing = True, do_not_queue = True)
        #s.bind((ip,int(port)))
        print "i'm running"
        time.sleep(10)
    except:
        print "another programme is running"

if __name__ == '__main__':
    check_dbus("name")

```



<a name="F"></a>

## 写成service并交由第三方托管

这个就没什么好说的了，将Python文件打包并执行，由supevisor等第三方工具来管理。




## 参考文章

[v2ex](https://www.v2ex.com/t/316723#reply21)

[stackoverflow-Preventing multiple process instances on Linux](http://stackoverflow.com/questions/2964391/preventing-multiple-process-instances-on-linux)

[Linux 编程中的文件锁之 flock](http://blog.jobbole.com/102538/)

[Linux 2.6 中的文件锁](https://www.ibm.com/developerworks/cn/linux/l-cn-filelock/)

[Python wiki](https://wiki.python.org/moin/DbusExamples)




***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***
