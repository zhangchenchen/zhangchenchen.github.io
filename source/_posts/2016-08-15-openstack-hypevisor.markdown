---
layout: post
title:  "openstack系列--服务器虚拟化知识"
date:   2016-08-15 11:06:25
tags: openstack
---


## 概念与认识

直接引用[wiki](https://zh.wikipedia.org/zh-cn/%E8%99%9B%E6%93%AC%E5%8C%96):

 > 在计算机技术中，虚拟化（英语：Virtualization）是一种资源管理技术，是将计算机的各种实体资源，如服务器、网络、内存及存储等，予以抽象、转换后呈现出来，打破实体结构间的不可切割的障碍，使用户可以比原本的配置更好的方式来应用这些资源。这些资源的新虚拟部分是不受现有资源的架设方式，地域或物理配置所限制。一般所指的虚拟化资源包括计算能力和数据存储。

关于虚拟化的大致分类，参考[虚拟化分类](https://segmentfault.com/a/1190000004347086)

目前比较常用的虚拟化技术包括KVM,XEN,vmware workstation等，本文重点介绍下KVM技术。

## KVM 技术

#### KVM 简介
KVM（Kernel-based Virtual Machine） 技术是在x86硬件平台（包含虚拟化扩展如Intel VT或AMD-V）上的一种完全虚拟化解决方案。效率可达到物理机的80％以上。
它包含一个为处理器提供底层虚拟化可加载的核心模块kvm.ko（kvm-intel.ko 或 kvm-AMD.ko)。
kvm还需要一个经过修改的QEMU软件（qemu-kvm），作为虚拟机上层控制和界面。
kvm能在不改变linux或windows镜像的情况下同时运行多个虚拟机，（ps：它的意思是多个虚拟机使用同一镜像）并为每一个虚拟机配置个性化硬件环境（网卡、磁盘、图形适配器……）。
在主流的linux内核，如2.6.20以上的内核均包含了kvm核心。

图片来自wikipedia
![kvm image](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/20160815-kvm-image.png)

#### 名词解释：KVM QEMU qemu-kvm Libvirt

[KVM](http://www.linux-kvm.org/page/Main_Page):一种虚拟化技术

[QEMU](http://wiki.qemu.org/Index.html):一种模拟器，它向Guest OS模拟CPU和其他硬件，Guest OS认为自己和硬件直接打交道，其实是同Qemu模拟出来的硬件打交道，Qemu将这些指令转译给真正的硬件。

[qemu-kvm](http://wiki.qemu.org/KVM):kvm负责cpu虚拟化+内存虚拟化，实现了cpu和内存的虚拟化，但kvm不能模拟其他设备。qemu模拟IO设备（网卡，磁盘等），kvm加上qemu之后就能实现真正意义上服务器虚拟化。因为用到了上面两个东西，所以称之为qemu-kvm。

[Libvirt](https://libvirt.org/index.html):目前使用最为广泛的一种对虚拟机进行管理的工具和API。Libvirtd是一个daemon进程，可以被本地的virsh调用，也可以被远程的virsh调用，Libvirtd调用qemu-kvm操作虚拟机。
![Libvirt arcitecture](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/20160815-Libvirt.jpg)

[更多Libvirt介绍](https://segmentfault.com/a/1190000004356172)

## KVM 与 [Docker](https://en.wikipedia.org/wiki/Docker_(software))

docker优势：
 - 更轻量，启动和关闭速度更快
 - 性能比虚拟化来说具有优势

docker劣势：
 - 安全性
 - 相对虚拟化来说生态环境还未成熟


[推荐文章](http://www.jianshu.com/p/ca7d5438ce1e)



## 参考文章

[OpenStack设计与实现（一）虚拟化](https://segmentfault.com/a/1190000004347086)

[kvm vs docker](http://bbs.chinaunix.net/thread-4233314-1-1.html)

[虚拟机已死 “容器”才是未来？](http://www.imooc.com/article/7303)

***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***
