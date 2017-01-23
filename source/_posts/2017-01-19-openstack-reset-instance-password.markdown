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


[采用 cloud-init 的方式](#E)


<a name="A"></a>

## 引出

- 要解决的问题很明确：就是如果虚拟机的连接采用用户名密码登录的方式，而密码忘记的话，需要采取什么手段解决。
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

## 采用 cloud-init 的方式

- 这种方法算是所有方法里面最轻便的，但坏处是需要自己定制脚本。对于cloud-init，不熟悉的话可以先翻一下[官方手册](http://cloudinit.readthedocs.io/en/latest/topics/capabilities.html)
- 原理很简单：借助cloud-init,在虚拟机启动的时候开启一个服务，用来监测metadata中设定的某个值，如果该值发生改变（或者满足其他条件）即做出密码更改的动作并reboot。
- 可喜的是，我在github找到了类似的代码[openstack-password-reset](https://github.com/vvaldez/openstack-password-reset) ,不过这个代码只是考虑了RH7系列，而且密码是随机生成的，如果再推给openstack，可能更复杂了。我又更改一下脚本，支持更多Linux版本，且把重设后的密码定死了。年后会把改过的代码挂到github上。
- 这里面的reset Python程序是通过外链获取的，于是干脆在nova里加了一个API，用来获取该程序。
- 如果是传递多个文件给cloud-init的话，需要使用MIME的格式，tips:一般是把多个脚本/cloud-config文件 打包成MIME格式文件，然后压缩成gzip格式，传给cloud-init。








## 参考文章

[Password Reset](https://github.com/vvaldez/openstack-password-reset)

[虚拟机系统密码的修改方案](http://niusmallnan.com/_build/html/_templates/openstack/inject_passwd.html#id2)

[CloudInit & User-Data](http://blog.csdn.net/heaven619/article/details/53420258)


***END***