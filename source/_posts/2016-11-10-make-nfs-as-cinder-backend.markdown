---
layout: post
title:  "openstack系列--设置nfs作为cinder backend"
date:   2016-11-10 13:06:25
tags: openstack
---

## 目录结构：

[nfs 设置](#A)

[修改cinder 配置文件 ](#B)



<a name="A"></a>

## nfs 设置

首先对nfs进行设置，大体以下几个步骤：

 - 磁盘分区，格式化磁盘
 ![fdisk](http://7xrnwq.com1.z0.glb.clouddn.com/20161110-fdisk.png)
 在格式化磁盘的时候出现如下错误：
 ![mkfswrong](http://7xrnwq.com1.z0.glb.clouddn.com/20161110-mkfswrong.png)
 搜了很长时间没找到答案，看提示应该是系统还有什么进程在使用这块磁盘，后请教别人后发现该磁盘之前做过ceph的osd盘，虽然已经删除了对应的关联，但是并没有删除完全，必须将/var/lib/ceph/osd/下对应的ceph包删除。
 ![ceph osd](http://7xrnwq.com1.z0.glb.clouddn.com/20161110ceph.png)
 然后，格式化成功。
 ![mkfs](http://7xrnwq.com1.z0.glb.clouddn.com/20161110mkfsright.png)

 - 新建共享目录，mount 磁盘
 ![mount file](http://7xrnwq.com1.z0.glb.clouddn.com/20161111mount-file.png)
 - nfs安装 先查看有没有安装
 ![nfs-install](http://7xrnwq.com1.z0.glb.clouddn.com/20161111nfs-install.png)
 - 配置nfs共享文件路径并启动nfsserver.
 ![nfs-config](http://7xrnwq.com1.z0.glb.clouddn.com/20161111nfs-config.png)

 
<a name="B"></a>

## 修改cinder 配置文件

 - 首先修改cinder配置文件cinder.conf，指定nfsshare文件路径(图中注释部分为之前的ceph配置) 
![cinder-config](http://7xrnwq.com1.z0.glb.clouddn.com/20161110cinder-config.png)

 - 添加nfsshare 文件，只需要指出nfs暴露的接口即可。
![nfsshare](http://7xrnwq.com1.z0.glb.clouddn.com/20161110-nfsshare.png)
 - 重启cinder-volume
 - 发现cinder volume报错，发现是cinder对挂载的nfs没有权限，修改nfs挂载文件的user,group 为cinder，或者直接chmod 777 (不安全)
 - 重启cinder-volume，成功。
 - 查看mount验证下。
 ![mount verify](http://7xrnwq.com1.z0.glb.clouddn.com/20161111-mount-nfs.png)


## 参考文章

[磁盘分区，格式化与检验](http://wiki.jikexueyuan.com/project/learn-linux-step-by-step/disk-partition-format-and-test.html)

[CentOS 6.5下NFS安装配置](http://www.centoscn.com/CentosSecurity/SoftSecurity/2015/0408/5118.html)
[Configure an NFS storage back end](http://docs.openstack.org/admin-guide/blockstorage-nfs-backend.html)


***END***
