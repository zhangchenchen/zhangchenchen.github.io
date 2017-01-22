---
layout: post
title:  "openstack系列--cinder 源码解读(一)---常见命令行操作"
date:   2016-12-23 09:06:25
tags: openstack
---

## 目录结构：

[cinder 的命令行操作](#A)




- 关于cinder不再赘述，之前文章由介绍。
- cinder 的 API 目前有三个版本，本次讨论以v2为基准。
- 因为版本不同，代码也可能不同，该版本为Openstack Liberty
- cinder 的命令行是由python-cinderclient 项目提供（已逐渐整合进python-openstackclient项目）
- 除此之外，还有一个cinder-manage 的命令行，主要是对cinder 各服务的管理，是由cinder项目本身cinder-manage模块提供。


<a name="A"></a>

## cinder 的命令行操作

直接命令行：cinder help ，查看一下subcommand 都包含哪些操作。
（本想用图片展示，画了半天，太麻烦了。）

- 关于 volume 的操作：
    1. create/delete/list/rename/show
    2. force-delete :不管state如何，强制删除
    3. migrate:迁移至另一主机
    4. set-bootable:设置为可启动盘
    5. reset-state:设置volume 的state
    6. upload-to-image : bootable 必须为True
    7. readonly-mode-update :设置为只读模式

- 关于 type 的操作：

    1. 操作如图： 
    ![type-op](http://7xrnwq.com1.z0.glb.clouddn.com/2016-12-23-cinder-volume-type-op.png) 
    2. volume type 还是比较重要的，后面qos的设置的时候就是与volume-type 绑定的。
    3. 还有一个encryption-type 也是 type相关，绑定instance之后，该磁盘上的内容会加密。
    ![encrv-type-op](http://7xrnwq.com1.z0.glb.clouddn.com/2016-12-23-encry-type-op.png)

- 关于 snapshot 的操作：

    1. 
    ![snapshot-op](http://7xrnwq.com1.z0.glb.clouddn.com/2016-12-23-cinder-snapshot-op.png)

- 关于 backup 的操作：
    1. ![backup-op](http://7xrnwq.com1.z0.glb.clouddn.com/2016-12-23-cinder-backup-op.png)
    2. backup 与 snapshot 的区别：1，snapshot 对于原volume是强依赖的，要想删除volume，必须先删除依赖他的snapshot，snapshot主要用于恢复数据到某个时间点，且原volume破损，那么对应的snapshot也不可用。而backup则对volume就不具有依赖性。所以即使备份相关的卷出现故障，还是可以恢复备份中数据。操作方法是创建一个新的空白卷，将备份restore到这个空白卷即可。2，snapshot 只能在同一存储区域（比如 ceph 里就是同一volume,backup 可以转存到别的地方）
    3. 补：发现如果是从snapshot 创建的volume，删除snapshot 之前要先删除依赖的volume，否则，删除不了。而且没有提示，不知道是否有bug，或是版本太低，下次用最新版试下。

- 关于 qos的操作：

    1. ![qos-op](http://7xrnwq.com1.z0.glb.clouddn.com/2016-12-23-qos-op.png)
    2. qos 要与volume type 一块使用。
    3. [深入理解Openstack QoS控制实现与实践](http://int32bit.me/2016/07/16/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3Openstack-QoS%E6%8E%A7%E5%88%B6%E5%AE%9E%E7%8E%B0%E4%B8%8E%E5%AE%9E%E8%B7%B5/)

- 关于 quota 的操作：
    1. ![quota-op](http://7xrnwq.com1.z0.glb.clouddn.com/2016-12-23-quota-op.png)

- 关于 transfer 的操作：
    1. ![transfer-op](http://7xrnwq.com1.z0.glb.clouddn.com/2016-12-23-transfer-op.png)
    2. 所谓 transfer 就是admin把 volume  从一个project 转到另一个project。

- 关于 service 的操作：
    1. ![service-op](http://7xrnwq.com1.z0.glb.clouddn.com/2016-12-23-volume-service-op.png)
    2. 看下service-list:
    ![service-list](http://7xrnwq.com1.z0.glb.clouddn.com/2016-12-28-cinder-service-list.png)
    3. enable/disable 即是对status的操作





## 参考文章

[深入理解Openstack QoS控制实现与实践](http://int32bit.me/2016/07/16/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3Openstack-QoS%E6%8E%A7%E5%88%B6%E5%AE%9E%E7%8E%B0%E4%B8%8E%E5%AE%9E%E8%B7%B5/)

[Volume Transfer(volume过户)](http://www.jianshu.com/p/fd432c29f277)

[openstack使用Ceph存储后端创建虚拟机快照原理剖析](http://int32bit.me/2016/10/25/Openstack%E4%BD%BF%E7%94%A8Ceph%E5%AD%98%E5%82%A8%E5%90%8E%E7%AB%AF%E5%88%9B%E5%BB%BA%E8%99%9A%E6%8B%9F%E6%9C%BA%E5%BF%AB%E7%85%A7%E5%8E%9F%E7%90%86%E5%89%96%E6%9E%90/)



***END***
