---
layout: post
title:  "ceph系列-- ceph 运维(二)-----常见TroubleShooting"
date:   2016-12-21 11:06:25
tags: ceph
---

## 目录结构：

[监视器MON故障排除](#A)

[OSD 故障排除](#B)

[归置组排障 ](#C)








<a name="A"></a>

## 监视器MON故障排除

 
![ceph-mon-trouble-shot](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/2016-12-21-ceph-mon-trouble-shotting.png)

简单记下修复 monmap的过程：

    1. 如果该mon节点是quorum 节点，直接获取monmap: ceph mon getmap -o /tmp/monmap
    2. 如果该mon节点不是quorum 节点，从别的quorum节点(假设有ID，且已stop)获取:ceph-mon -i ID --extract-monmap /tmp/monmap
    3. stop 你要修复的mon节点的mon进程
    4. 注入monmap:ceph-mon -i ID --inject-monmap /tmp/monmap
    5. start monitor




<a name="B"></a>

## OSD 故障排除

- 收集 OSD 数据

    1. 查看ceph日志：/var/log/ceph 
    2. 利用管理套接字：ceph daemon {socket-file} help （可以获取在运行时配置，历史操作，操作的优先队列状态，在进行的操作，列出性能计数器）
    3. 查看空间占用：df -h
    4. I/O 统计信息: iostat -x
    5. 诊断消息: dmesg | grep scsi

- 停止自动重均衡

    1. 如果不想在停机维护 OSD 时让 CRUSH 自动重均衡，提前设置 noout ：ceph osd set noout
    2. 停机维护失败域内的 OSD 
    3. 维护结束后，重启OSD。
    4. 解除 noout 标志：ceph osd unset noout

- OSD 未成功运行
通常情况下，简单地重启 ceph-osd 进程就可以重回集群并恢复。
    1. OSD  运行不起来：检查配置文件（语法错误），或配置路径错误。查看是否超过系统默认最大线程数，以及与内核版本是否冲突。
    2. OSD down: 查看日志是否因为硬盘错误

- OSD 龟速或无响应
 
 有很多原因会导致此问题：

    1. 网络问题：netstat -s
    2. 驱动器配置:一个存储驱动器应该只用于一个 OSD 。
    3. 坏扇区和碎片化硬盘.
    4. 监视器和 OSD 在同一主机：监视器是普通的轻量级进程，但它们会频繁调用 fsync() ，这会妨碍其它工作量，特别是监视器和 OSD 共享驱动器时。
    5. 日志记录级别：如果你为追踪某问题提高过日志级别、但结束后忘了调回去，这个 OSD 将向硬盘写入大量日志。如果你想始终保持高日志级别，可以考虑给默认日志路径挂载个硬盘。
    6. 内核与 SYNCFS 问题：试试在一主机上只运行一个 OSD ，看看能否提升性能。老内核未必支持有 syncfs(2) 系统调用的 glibc 。
    7. 文件系统问题：推荐基于 xfs 或 ext4 部署集群。
    8. 内存不足：建议为每 OSD 进程规划 1GB 内存。
    9. OLD REQUESTS 或 SLOW REQUESTS：如果某 ceph-osd 守护进程对一请求响应很慢，它会生成日志消息来抱怨请求耗费的时间过长。


- 一会up一会down 的OSD

即俗称的打摆子。
网络部署时建议同时部署公网（前端）和集群网（后端），这样能更好地满足对象复制的容量需求。另一个优点是你可以运营一个不连接互联网的集群，以此避免拒绝攻击。 OSD 们互联和检查心跳时会优选集群网（后端），如果集群网（后端）失败、或出现了明显的延时，同时公网（前端）却运行良好， 这时 OSD 们会向监视器报告邻居 down 了、同时报告自己是 up 的，我们把这种情形称为打摆子（ flapping ）。可以强制OSD停止标记：

```bash
ceph osd set noup      # prevent OSDs from getting marked up
ceph osd set nodown    # prevent OSDs from getting marked down
```

下列命令可清除标记：


```bash
ceph osd unset noup
ceph osd unset nodown
```



<a name="C"></a>

## 归置组排障

在我们查看ceph 状态的时候，通常会看到pgmap的状态：active+clean。这是正常状态。不过也有不正常的时候。

- pg 状态为 inactive-----归置组长时间无活跃（即它不能提供读写服务了）：OSD 的互联心跳检测失败，标记为down,重启对应osd进程。
- pg 状态为unclean------归置组长时间不干净（例如它未能从前面的失败完全恢复）：找不到对象。
- pg 状态为stale-----归置组状态没有被 ceph-osd 更新，表明存储这个归置组的所有节点可能都挂了。

详参[归置组排障](http://docs.ceph.org.cn/rados/troubleshooting/troubleshooting-pg/)





## 参考文章

[监视器故障排除](http://docs.ceph.org.cn/rados/troubleshooting/troubleshooting-mon/)

[OSD 故障排除](http://docs.ceph.org.cn/rados/troubleshooting/troubleshooting-osd/)

[归置组排障](http://docs.ceph.org.cn/rados/troubleshooting/troubleshooting-pg/)

[日志记录和调试](http://docs.ceph.org.cn/rados/troubleshooting/log-and-debug/)


***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***
