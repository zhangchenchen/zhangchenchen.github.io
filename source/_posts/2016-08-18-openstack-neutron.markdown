---
layout: post
title:  "openstack系列--neutron"
date:   2016-08-22 14:36:25
tags: openstack
---





<a name="A"></a>

## 概述 

In short，neutron就是openstack为实现云平台上networking as a service而开发的项目。可以简单地理解为网络部分的虚拟化由neutron实现。


<a name="B"></a>

## 核心概念解释 


 1, Tenant networks

 首先解释下network,一个network就是一个二层独立广播域。
 由租户在project中创建的network.默认创建的network是完全独立的，不与其他project共享。neutron提供的tenant networks包括如下几种类型：

  - Flat : 所有的虚拟机实例都在同一个network中，没有分隔策略,都在一个广播域。基本不用这种类型。

  - Vlan : 提供对Vlan的支持。 

  - GRE and VXLAN ：利用隧道技术封装二层协议帧在L3或L4传输，实现两个虚拟Vlan的二层联通。

 2, Provider networks

 由openstack administrator 创建的network，在数据中心中映射物理网络。通常的类型为flat或Vlan

 ![Provider networks](http://7xrnwq.com1.z0.glb.clouddn.com/20160819NetworkTypes.png)

 3, Subnet 

  subnet属于网络中的三层概念，指定一段IPV4或者IPV6地址并描述其相关的配置信息。它附加在一个二层network 上指明属于这个network的虚拟机可使用的IP范围。
 
 4, Port

 port 是一个设备的连接点，例如一台虚拟机的虚拟网卡。同时描述了相关的网络配置，例如该端口的MAC地址与IP地址。

 ![部分核心概念](http://7xrnwq.com1.z0.glb.clouddn.com/20160819neutron_core_entities.png)

 5, [Vlan VS Vxlan](https://ask.openstack.org/en/question/51388/whats-the-difference-between-flat-gre-and-vlan-neutron-network-types/)



<a name="C"></a>

## neutron 提供的服务

 1, 基础的网络虚拟化（core service）
  上文提到的network，subnet等都是一些基础的网络虚拟化服务。 

 2, DHCP 服务， 路由服务 
  这两个服务 都需要network namespace 技术来实现。

 3, 安全组服务
  通过 Linux Iptables实现。

 4, LBaas
  它允许租户动态的在自己的网络创建一个负载均衡设备。

 5, FWaas
  防火墙一般放在网关上，用来隔离子网之间的访问。因此，防火墙即服务（FireWall as a Service）也是在网络节点上（具体说来是在路由器命名空间中）来实现。
  目前，OpenStack 中实现防火墙还是基于 Linux 系统自带的 iptables，所以大家对于其性能和功能就不要抱太大的期望了。
  一个可能混淆的概念是安全组（Security Group），安全组的对象是虚拟网卡，由L2 Agent来实现，比如neutron_openvswitch_agent 和 neutron_linuxbridge_agent，会在计算节点上通过配置 iptables 规则来限制虚拟网卡的进出访问。防火墙可以在安全组之前隔离外部过来的恶意流量，但是对于同个子网内部不同虚拟网卡间的通讯不能过滤（除非它要跨子网）。
  可以同时部署防火墙和安全组实现双重防护。
 
 6, DVR(分布式路由)
  按照 Neutron 原先的设计，所有网络服务都在网络节点上进行，这意味着大量的流量和处理，给网络节点带来了很大的压力。
  这些处理的核心是路由器服务。任何需要跨子网的访问都需要路由器进行路由。
  很自然，能否让计算节点上也运行路由器服务？这个设计思路无疑是更为合理的，但具体实施起来需要诸多细节上的技术考量。
  为了降低网络节点的负载，同时提高可扩展性，OpenStack 自 Juno 版本开始正式引入了分布式路由（Distributed Virtual Router，DVR）特性（用户可以选择使用与否），来让计算节点自己来处理原先的大量东西向流量和非 SNAT  南北流量（有 floating IP 的 vm 跟外面的通信）。
  这样网络节点只需要处理占到一部分的 SNAT （无 floating IP 的 vm 跟外面的通信）流量，大大降低了负载和整个系统对网络节点的依赖。很自然的，FWaaS 也可以跟着放到计算节点上。
  DHCP 服务、VPN 服务目前仍然需要集中在网络节点上进行。

 7, VPNaas



<a name="D"></a>

## 架构与原理

![neutron架构图](http://7xrnwq.com1.z0.glb.clouddn.com/20160819-NEUTRON-ARCHITECTURE.png)

Neutron，其实和其他的OpenStack组件差不多，他都是一个中间层，自己基本不干具体的活，通过插件的机制，调用第三方的组件来完成相关的功能。



## 参考文章

[Openstack Neutron: Introduction](http://abregman.com/2016/01/04/openstack-neutron-introduction/)

[OpenStack Neutron网络的肤浅理解](http://www.chenshake.com/openstack-superficial-understanding-of-neutron-network/)

[Overview and components](http://docs.openstack.org/mitaka/networking-guide/intro-os-networking-overview.html#openstack-networking-concepts)

[深入理解Neutron -- OpenStack 网络实现](https://www.gitbook.com/book/yeasy/openstack_understand_neutron/details)

[openstack doc](http://docs.openstack.org/mitaka/networking-guide/intro-os-networking-overview.html)

[openstack 设计与实现](https://www.amazon.cn/Open-Stack%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0-%E8%8B%B1%E7%89%B9%E5%B0%94%E5%BC%80%E6%BA%90%E6%8A%80%E6%9C%AF%E4%B8%AD%E5%BF%83/dp/B00WG4WJ9I/ref=sr_1_1?ie=UTF8&qid=1471599232&sr=8-1&keywords=openstack+%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0)

***END***
