---
layout: post
title:  "linux-- 常见的linux 服务器性能监控工具"
date:   2017-04-06 10:12:32
tags: 
  - linux
  - performance monitor
---



## 引出

性能监控是服务器运维非常重要的一部分，本文记录非常好用的几个性能监控工具，算是一个小的总结。
本文组织方式是按cpu,mem(内存)，网络，磁盘io等具体性能监控的方向分类，末尾介绍一些相对大而全，涵盖各方面的企业级监控工具。。


## cpu/mem 监控工具


### top

这应该是第一个想到的性能监控命令，示例截图如下：

![top](http://7xrnwq.com1.z0.glb.clouddn.com/2017-04-06-top.png)

我们关注得点应该包括：cpu,mem负载，load-average，以及占用内存或cpu最多的进程。
该命令还有一些交互的子命令，比如 "c" ,是按照CPU 占用排序，"m"是按照内存占用排序等。 

### sar 

该命令号称系统监控的瑞士军刀，目前Linux上最为全面的系统性能分析工具之一，可以从14个大方面对系统的活动进行报告，包括文件的读写情况、系统调用的使用情况、串口、CPU效率、内存使用状况、进程活动及IPC有关的活动等，使用也是较为复杂。
sar 默认显示的是从零点开始每隔十分钟到现在的CPU情况，如果是查看之前的报告，需要指定日志报告，sar -f /var/log/sysstat/sa25 。

![sar-u](http://7xrnwq.com1.z0.glb.clouddn.com/2017-04-06-sar.png)

解释下各列的指标：
- %user 用户模式下消耗的CPU时间的比例；
- %nice 通过nice改变了进程调度优先级的进程，在用户模式下消耗的CPU时间的比例
- %system 系统模式下消耗的CPU时间的比例；
- %iowait CPU等待磁盘I/O导致空闲状态消耗的时间比例；
- %steal 利用Xen等操作系统虚拟化技术，等待其它虚拟CPU计算占用的时间比例；
- %idle CPU空闲时间比例；

查看 内存使用情况：sar -r 

![sar-r](http://7xrnwq.com1.z0.glb.clouddn.com/2017-04-06-dar-r.png)

查看内存页面交换发生状况：sar -W

![sar-w](http://7xrnwq.com1.z0.glb.clouddn.com/2017-04-06-sar-w.png)

查看带宽信息：sar -n DEV

![sar-n](http://7xrnwq.com1.z0.glb.clouddn.com/2017-04-06-SARN.png)



要判断系统瓶颈问题，有时需几个 sar 命令选项结合起来；
- 怀疑CPU存在瓶颈，可用 sar -u 和 sar -q 等来查看
- 怀疑内存存在瓶颈，可用sar -B、sar -r 和 sar -W 等来查看
- 怀疑I/O存在瓶颈，可用 sar -b、sar -u 和 sar -d 等来查看

更详细的命令参数可以参考[10 Useful Sar (Sysstat) Examples for UNIX / Linux Performance Monitoring](http://www.thegeekstuff.com/2011/03/sar-examples/)

## 网络监控工具


### netstat

该命令会显示各种与网络相关的信息，比如网络连接，路由表，接口状态等。

![netstst](http://7xrnwq.com1.z0.glb.clouddn.com/2017-04-06-netstat.png)

解释下主要有两个部分：

- Active Internet connections，称为源TCP连接，其中"Recv-Q"和"Send-Q"指接收队列和发送队列。这些数字一般都应该是0。如果不是则表示软件包正在队列中堆积
- Active UNIX domain sockets，称为有源Unix域套接口(跟网络套接字一样，但是只能用于本机通信，性能可以提高一倍)。Proto显示连接使用的协议,RefCnt表示连接到本套接口上的进程号,Types显示套接口的类型,State显示套接口当前的状态,Path表示连接到套接口的其它进程使用的路径名。
常用参数如下：
- a (all)显示所有选项，默认不显示LISTEN相关
- t (tcp)仅显示tcp相关选项
- u (udp)仅显示udp相关选项
- n 拒绝显示别名，能显示数字的全部转化成数字。
- l 仅列出有在 Listen (监听) 的服務状态

### iptraf

非常实用的tcp/udp网络监控工具，有一个非常简洁的界面，常用的功能包括：

- IP流量监控器，用来显示网络中的IP流量变化信息。包括TCP标识信息、包以及字节计数，ICMP细节，OSPF包类型。
- 简单的和详细的接口统计数据，包括IP、TCP、UDP、ICMP、非IP以及其他的IP包计数、IP校验和错误，接口活动、包大小计数。
- TCP和UDP服务监控器，能够显示常见的TCP和UDP应用端口上发送的和接收的包的数量。
局域网数据统计模块，能够发现在线的主机，并显示其上的数据活动统计信息。
- TCP、UDP、及其他协议的显示过滤器，允许你只查看感兴趣的流量。

以下为一个查看网卡统计信息的界面：

![iptraf](http://7xrnwq.com1.z0.glb.clouddn.com/2017-04-06-iptraf.png)



## 大而全的企业级性能监控工具

目前开源的企业级性能监控工具应用比较广泛的应该属Nagios 与Zabbix了，关于两者对比，可以参考：

[Zabbix vs Nagios vs PandoraFMS: an in depth comparison](https://blog.pandorafms.org/zabbix-vs-nagios-vs-pandorafms-an-in-depth-comparison/)

[开源监控系统中 Zabbix 和 Nagios 哪个更好？](https://www.zhihu.com/question/19973178)


## 参考文章

[10 Useful Sar (Sysstat) Examples for UNIX / Linux Performance Monitoring](http://www.thegeekstuff.com/2011/03/sar-examples/)

[你需要知道的16个Linux服务器监控命令](http://blog.jobbole.com/15430/)

[linuxtools-rst](http://linuxtools-rst.readthedocs.io/zh_CN/latest/tool/sar.html)

[iptraf：一个实用的TCP/UDP网络监控工具](https://linux.cn/article-5430-1.html)


***END***
