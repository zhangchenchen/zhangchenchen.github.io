---
layout: post
title:  "Openstack-- nova-network 网络实现一窥"
date:   2017-06-30 09:16:11
tags: 
  - openstack
  - nova
---

## 引出

nova-network 是Neutron（或Quantum）之前，openstack 的网络管理项目，随着Neutron项目的愈发成熟，nova-network 也随之逐渐被弃用。
不过，其实 nova-network 还是非常好用的，相对neutron 来说，简单轻便，也经受过生产环境的验证。而且，可以通过简单配置将虚拟机二层网络与真实物理网络打通，实现利用fixed-ip就可以直接访问虚拟机的单网络环境。
本文主要总结下 nova-network 的网络管理以及实现原理，并与neutron 做一下比较。

##  nova-network 网络类型

nova-network 支持两种网络类型（确切地说是三种，这里将Flat与FlatDHCP看做一类），分别是Flat/FlatDHCP 和 Vlan类型。

### FlatManager and FlatDHCPManager

顾名思义，flat即扁平化，也就是说所有的虚拟机都在一个大的二层网络空间内，这种网络模式一般生产环境中很少用，多用于POC阶段。
关于flat networking，我在openstack 上找到一篇[wiki](https://wiki.openstack.org/wiki/UnderstandingFlatNetworking)，讲解的比较清楚。可惜wiki 上只有讲解flat的，没有其他网络类型。

这里借上文中的一张图简单说下flat这种网络类型，以多节点，多网卡为例：

![multi-host-1](http://oeptotikb.bkt.clouddn.com/2017-06-30-FlatNetworkMultInterface.png)

由图看出，计算节点内都有一个网桥br100与 物理网卡eth0相连，计算节点内的虚拟机通过该网桥与控制节点的对应物理网卡连接实现通信。
虚拟机实现南北通信大致如下图：

![multi-host-2](http://oeptotikb.bkt.clouddn.com/2017-06-30-MultiInterfaceOutbound_2.png)

flatDHCP 其实就是在计算节点再运行一个dnsmasp来提供DHCP服务，就不需要借助外部的DHCP服务，两者对比大致如下：

flat networking :

![flat](http://oeptotikb.bkt.clouddn.com/2017-06-30-flatdhcp.png)

flatDHCP networking:

![flat-dhcp](http://oeptotikb.bkt.clouddn.com/QQ%E6%88%AA%E5%9B%BE20170630144105.png)


### VlanManager

flat networking 有一个很大的缺点就是所有虚拟机都在一个二层网络，没有租户间的网络隔离，不够灵活，且广播风暴的影响太大，这种情况下，vlan 就出现了。

vlan 网络或给每个租户创建一个（或多个）二层网络，且这些二层网络相互隔离。不过这种网络类型需要支持vlan tag的交换机才可以使用。

## nova-network 网络实现原理

简单介绍下各自的实现原理

### Flat/FlatDHCP网络实现原理

Flat类型的基本上上文已经讲了，主要看下flatDHCP 的实现：

![flat-dhcp](http://oeptotikb.bkt.clouddn.com/2017-06-30-flat-dhcp-real.png)

跟flat类似，不过在计算节点的网桥上会有一个dnsmasq进程监听并实现该节点的虚拟机IP动态分配。
还有一点要注意的是，不同计算节点的虚拟机内网关也不同，比如，上图中，vm_1与vm_2的网关都是10.10.0.1，但在右边计算节点的vm_3 和vm_4的网关却是10.10.0.4。

### Vlan实现原理

一图胜千言：

![multi-host-vlan](http://oeptotikb.bkt.clouddn.com/2017-06-30-vlanmanager-2-hosts-2-tenants.png)

上面这幅图就是vlan模式部署最为广泛的场景，即multi-node模式，也就是说不光是网络节点需要运行nova-network服务，计算节点也要运行nova-network（还需要运行nova-api以提供metadata服务），这样就避免出现SPOF,也可以达到分流的作用（跟neutron 的DVR很类似，估计DVR就是借鉴了这里）。
需要注意的是每个vlan网桥都是租户独占的，且会创建vlan接口如vlan102,依据802.1q协议打vlanid，与网关eth0连接。Dnsmasq监听网桥网关，负责fixedip的分配。switch port设定为chunk mode。eth0负责vm之间的数据通信，另一网卡如eth1负责外网访问。上图只是显示了一个网卡，如下图是一个多网卡的情况：

![vlan-multi-host](http://oeptotikb.bkt.clouddn.com/2017-06-30-multi-host-vlan.png)


## nova-network vs neutron

之前总结过一篇关于[neutron 实现二三层网络的总结](https://zhangchenchen.github.io/2017/02/12/neutron-layer2-3-realization-discovry/),就功能性来说，nova-network 只支持两种网络类型，neutron增加了overlay network 的支持，如vxlan/gre, 突破了vlan id个数限制从而支持更多数目的二层网络。而且nova-network 即使是不同租户间，也不允许使用相同的网段，因为租户网络是通过ip+vlan的形式来区分，neutron 就完全可以。
但也不是nova-network 就一文不取，相比于neutron网络，虽说没有neutron那么多的功能插件，仅有bridge，但是其稳定性已得到大多数用户的验证，对于小规模的私有云(1千台虚机的规模)，nova-network是可以考虑的。



## 参考文章


[OpenStack Networking – FlatManager and FlatDHCPManager](https://www.mirantis.com/blog/openstack-networking-flatmanager-and-flatdhcpmanager/)

[Openstack Networking for Scalability and Multi-tenancy with VlanManager](https://www.mirantis.com/blog/openstack-networking-vlanmanager/)

[Network service - easy version - nova-network](https://github.com/gc3-uzh-ch/gridka-school/blob/master/tutorial/nova_network.rst)

[Networking with nova-network](https://docs.openstack.org/admin-guide/compute-networking-nova.html)

[openstack 网络架构 nova-network + neutron](http://blog.csdn.net/beginning1126/article/details/41172365)

[UnderstandingFlatNetworking](https://wiki.openstack.org/wiki/UnderstandingFlatNetworking)

***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***