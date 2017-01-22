---
layout: post
title:  "openstack系列--重设虚拟机实例密码几种方法"
date:   2017-01-19 14:57:25
tags: openstack
---

## 目录结构：

[引出](#A)


[采用 nova get-password 方式](#B)

[采用 libvirt-set-admin-password ](#C)

[采用 nova rebuild instance 的方式](#D)


[]()


<a name="A"></a>

## 引出

- 要解决的问题很明确：就是如果虚拟机的连接采用用户名密码登录的方式，而密码忘记的话，需要采取什么手段解决。
- 最简单的解决方案当然是运营端通过VNC连接直接重设密码就行了。这里只讨论不进入虚拟机，依靠openstack提供的接口实现的情况。。
- 其实解决方案是要取决于真实的生产环境，虚拟化方式的不同，初始化虚拟机密码方式的不同，openstack版本的不同，都会造成某个方案的可行不可行。以下几种方案可能或多或少会出现无法实现的情况，楼主尽量把条件讲清楚。


<a name="B"></a>

## 采用 nova get-password 方式

- 利用nova 提供的这个接口可以获取instance的password,就不用密码reset了。
- 适用条件：虚拟化方式为XEN，不支持libvirt.


<a name="C"></a>

## 采用 libvirt-set-admin-password 

- Openstack L 版本新加入的功能，直接使用 "nova set-password "(或早期版本client的"nova root-password") 就可以，之前的版本该命令不支持Libvirt,仅支持XEN。
- 适用条件：Openstack Libvirt+ 版本，宿主机libvirt版本1.2.16+，虚拟机镜像安装2.3+ 版本的qemu-guest-agent，等等，限制条件比较多，详见[虚拟机系统密码的修改方案¶](http://niusmallnan.com/_build/html/_templates/openstack/inject_passwd.html#id2)               
<a name="D"></a>

## 采用 nova image-create / nova rebuild的方式

- 如果虚拟机是根据user-data来设定初始密码的，那么cloud-init只在第一次创建虚拟机执行一次，以后不会执行（reboot也不会执行）。那么我们也只能再次launch一下，方法如下。
- 首先对当前虚拟机做一次snapshot.

 ![image-create](http://7xrnwq.com1.z0.glb.clouddn.com/2017-01-22-image-create.png)

 ![image-snap](http://7xrnwq.com1.z0.glb.clouddn.com/2017-01-20-image-snap.png)

- 利用该snapshot ，设定好user-data重新boot 一个新的虚拟机

 ![snap-boot](http://7xrnwq.com1.z0.glb.clouddn.com/2017-01-20-snap-boot.png)

- 注意：此处只是保证系统盘数据是不变的，如果是数据盘的话还要将对应的数据盘detach再attch到新建的虚拟机中。当然，如果虚拟机是直接用的adminPass的话（即injectPassword的方式）也可以直接利用rebuild命令（rebuild只能用于image启动的instance,而不能用于volume 启动的instance）。

- 这种方法其实比较笨拙，不到万不得已一般不会这么做。


<a name="E"></a>

## 采用 






















## 参考文章

[Password Reset](https://github.com/vvaldez/openstack-password-reset)

[虚拟机系统密码的修改方案](http://niusmallnan.com/_build/html/_templates/openstack/inject_passwd.html#id2)

[CloudInit & User-Data](http://blog.csdn.net/heaven619/article/details/53420258)


***END***
