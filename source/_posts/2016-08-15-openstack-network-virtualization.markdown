---
layout: post
title:  "openstack系列--网络虚拟化基础知识"
date:   2016-08-15 17:06:25
tags: openstack
---



<a name="A"></a>

## 网络发展历程 

详见： 
[SDN软件定义网络从入门到精通》导论课](http://mp.weixin.qq.com/s?__biz=MjM5MTM3MzIzMg==&mid=209513316&idx=1&sn=e5dbd9a2ccccb88d0ee5c4d5790699c1#rd)

总结一下目前还常用的硬件设备：

 - 集线器(hub)：属于物理层产品，对信息进行中继和放大，从任意接口收到的数据会往其他所有接口广播。
 - 交换机(switch):链路层产品，记录终端主机的mac地址并生成mac表，根据mac表转发主机之间的数据流。能够进行VLAN隔离。
 - 路由器(router):网络层产品，基于IP寻址，采用路由表实现数据转发。


<a name="B"></a>

## 网络虚拟化与SDN

SDN(Software Define Network)是一种集中控制的网络架构，可将网络划分为数据层面和控制层面,它是实现网络虚拟化的一种解决方案。通常来说，只要网络硬件可以集中式软件管理，可编程化，控制转发层面分开，则可以认为这个网络是一个SDN网络。openstack 的neutron组件就是一种SDN 架构(其实，SDN的创始人就是Openstack Quantum
,即后来的neutron 的发起者之一)。SDN大体架构如下(图片来自[SDN与网络虚拟化的起源与现状](http://blog.sciencenet.cn/blog-1225851-974868.html))：

![SDN 大体架构](http://7xrnwq.com1.z0.glb.clouddn.com/20160816-SDN.jpeg)

网络虚拟化可以认为是一种网络技术，它可以在物理网络上虚拟多个相互隔离的虚拟网络，从而使得不同用户之间使用独立的网络资源切片，从而提高网络资源利用率，实现弹性的网络。网络虚拟化是云计算和SDN发展到一定阶段的产物，因此可以认为网络虚拟化是新一代的SDN。
延伸阅读：[SDN与网络虚拟化的起源与现状](http://blog.sciencenet.cn/blog-1225851-974868.html)

<a name="C"></a>

## openstack相关的网络虚拟化知识

 > Linux Bridge:Linux Bridge 是 Linux 上用来做 TCP/IP二层协议交换的设备，其功能大家可以简单的理解为是一个二层交换机或者Hub。多个网络设备可以连接到同一个 Linux Bridge，当某个设备收到数据包时，Linux Bridge 会将数据转发给其他设备。

![Linux bridge](http://7xrnwq.com1.z0.glb.clouddn.com/20160816-linux-bridge.png)

如上图，两个虚拟机，各自分配一个虚拟网卡vnet0,vnet1.他们连接在一个叫做br0 的Linux bridge 上。宿主机的网卡eth0分配到br0下，这样VM1与VM2可以相互通信，与外网也可以通信。



LAN 与VLAN :

LAN: 表示 Local Area Network，本地局域网，通常使用 Hub 和 Switch 来连接 LAN 中的计算机。一个 LAN 表示一个广播域。 其含义是：LAN 中的所有成员都会收到任意一个成员发出的广播包。 

VLAN: Virtual LAN。一个带有 VLAN 功能的switch 能够将自己的端口划分出多个 LAN。计算机发出的广播包可以被同一个 LAN 中其他计算机收到，但位于其他 LAN 的计算机则无法收到。 简单地说，VLAN 将一个交换机分成了多个交换机，限制了广播的范围，在二层将计算机隔离到不同的 VLAN 中。

VLAN的实现：一种方法是使用两个交换机，不同VLAN组的机器分别接到一个交换机。 另一种方法是使用一个带 VLAN 功能的交换机，将不同VLAN组的机器分别放到不同的 VLAN 中。 
请注意，VLAN 的隔离是二层上的隔离，不同VLAN无法相互访问指的是二层广播包（比如 arp）无法跨越 VLAN 的边界。但在三层上（比如IP）是可以通过路由器让 不同VLAN互通的。

KVM环境下VLAN的实现：

![利用Linux bridge 实现VLAN](http://7xrnwq.com1.z0.glb.clouddn.com/20160816-KVM-VLAN.jpg)

eth0 是宿主机上的物理网卡，有一个命名为 eth0.10 的子设备与之相连。 eth0.10 就是 VLAN 设备了，其 VLAN ID 就是 VLAN 10。 eth0.10 挂在命名为 brvlan10 的 Linux Bridge 上，虚机 VM1 的虚拟网卡 vent0 也挂在 brvlan10 上。这样，宿主机用软件实现了一个交换机（当然是虚拟的），上面定义了一个 VLAN10。 eth0.10，brvlan10 和 vnet0 都分别接到 VLAN10 的 Access口上。而 eth0 就是一个 Trunk 口。VM1 通过 vnet0 发出来的数据包会被打上 VLAN10 的标签。brvlan20 同理。


也可以利用 OpenVSwitch实现（官方推荐）

OpenVSwitch:

简称OVS,是一个高质量的、多层虚拟交换机，使用开源Apache2.0许可协议，由Nicira Networks开发，主要实现代码为可移植的C代码。它的目的是让大规模网络自动化可以通过编程扩展,同时仍然支持标准的管理接口和协议（例如NetFlow, sFlow, SPAN, RSPAN, CLI, LACP, 802.1ag）。此外,它被设计位支持跨越多个物理服务器的分布式环境，类似于VMware的vNetwork分布式vswitch或Cisco Nexus 1000 V。Open vSwitch支持多种linux 虚拟化技术，包括Xen/XenServer， KVM和irtualBox。

Linux nameSpace:

Linux 提供的一种内核级别环境隔离的方法。类似chroot系统调用（通过修改根目录把用户监禁到一个特定目录下，chroot内部的文件系统无法访问外部的内容），Linux Namespace在此基础上，提供了对UTS、IPC、mount、PID、network、User等的隔离机制。在openstack neutron 中就利用network namespace 进行网络的隔离。 


## 参考文章

[SDN与网络虚拟化的起源与现状](http://blog.sciencenet.cn/blog-1225851-974868.html)

[SDN软件定义网络从入门到精通》导论课](http://mp.weixin.qq.com/s?__biz=MjM5MTM3MzIzMg==&mid=209513316&idx=1&sn=e5dbd9a2ccccb88d0ee5c4d5790699c1#rd)

[Neutron 理解 (1): Neutron 所实现的虚拟化网络](http://geek.csdn.net/news/detail/67824)

[动手实践 Linux VLAN - 每天5分钟玩转 OpenStack（13）](https://www.ibm.com/developerworks/community/blogs/132cfa78-44b0-4376-85d0-d3096cd30d3f/entry/%E5%8A%A8%E6%89%8B%E5%AE%9E%E8%B7%B5_Linux_VLAN_%E6%AF%8F%E5%A4%A95%E5%88%86%E9%92%9F%E7%8E%A9%E8%BD%AC_OpenStack_13?lang=en)

[openstack doc](http://docs.openstack.org/mitaka/networking-guide/intro-os-networking-overview.html)

[OVS初级教程：使用open vswitch构建虚拟网络](http://www.sdnap.com/sdnap-post/3520.html)

[Introducing Linux Network Namespaces](http://blog.scottlowe.org/2013/09/04/introducing-linux-network-namespaces/)

[Docker基础技术：Linux Namespace（上）](http://coolshell.cn/articles/17010.html)

[Separation Anxiety: A Tutorial for Isolating Your System with Linux Namespaces](https://www.toptal.com/linux/separation-anxiety-isolating-your-system-with-linux-namespaces)

***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***
