---
layout: post
title:  "openstack-- neutron 二/三层网络实现探究"
date:   2017-02-12 17:44:25
tags: openstack
---



## 引出

- Neutron 是openstack 中提供网络虚拟化的组件，根据二层网络的实现方式不同(即agent的不同)，可以分为Linux bridge的方式，Openvswitch的方式。而且，lay2 network分为local,flat,vlan,vxlan 等类型(gre与vxlan类似，不再考虑)，本文就分析两种实现方式在这四种网络中的具体实现异同。最后会分析lay3 网络的实现异同。
- 本文内容主要来自[每天5分钟玩转 OpenStack](https://www.ibm.com/developerworks/community/blogs/132cfa78-44b0-4376-85d0-d3096cd30d3f?lang=en),图片也是来自该博客，本文仅是对其总结对比并作记录。


## lay2 local 型网络

生产环境中，local与flat网络是不会被使用的,vlan与vxlan是使用比较多的layer2网络。因为local网络只支持在同一宿主机的虚拟机互联，而flat网络会每个网络独占宿主机的一个物理接口，这在现实世界中是不允许的。但是vlan与vxlan的实现都是在local,flat网络的基础上实现的，所以还是有必要看一下这两种类型的网络的实现的。


### linux bridge 实现local 型网络

对于每个local network，ML2 linux-bridge agent 会创建一个bridge,instance的tap设备连接到该bridge。位于同一local network的instance会连接同一个bridge，这样instance之间就可以通信了。但因为bridge没有与宿主机物理网卡相连，所以跟宿主机无法通信，也没法与宿主机之外的其他机器通信。下图为示例：
![local-by-linux-bridge](http://7xrnwq.com1.z0.glb.clouddn.com/local-by-linux-bridge.jpg)

- 图中创建了两个local network,对应两个网桥。
- VM0 与 VM1 在同一个 local network中，它们之间可以通信.
- VM2 位于另一个 local network，无法与 VM0 和 VM1 通信。
- 两个local network都有自己的DHCP server,由dnsmasq在各自独立的net namespace实现，通过tap设备挂接在各自network的bridge上。



### openvswitch 实现local 型网络

ovs的实现要相对复杂些，ovs会自动创建三个ovs bridge，br-ex,br-int和br-tun，看名字大概也可以猜出，br-ex是用于外部网络，br-int是内部虚拟机的网络，br-tun是用于overlay network,也就是vxlan类型的网络会用到该bridge。local network 只需考虑br-int bridge。示例图如下：

![local-by-ovs](http://7xo6kd.com1.z0.glb.clouddn.com/upload-ueditor-image-20161229-1483015852547009927.jpg)

- 图中每个instance并不是跟之前一样通过tap设备直接挂载在ovs的 br-int上，而是通过一个新建的linux bridge以及一对veth pair跟br-int相连，为什么这样做呢，因为Open vSwitch 目前还不支持将 iptables 规则放在与它直接相连的 tap 设备上，这样就实现不了Security group，所以就加了一个linux bridge以支持iptables。
- 两个local network都有自己的DHCP server,由dnsmasq在各自独立的net namespace实现，通过tap设备挂接在各自network的bridge上。
- 都挂载在同一个br-int上，如何区分不同的local network呢，其实，br-int上挂载的虚拟网卡或DHCP对应的port都有一个特殊的tag属性。同一网络的tag相同，不同网络的tag不同，这里跟vlan的vlan ID类似，同一tag属性的port是二层连通的。
![port-tag](http://7xo6kd.com1.z0.glb.clouddn.com/upload-ueditor-image-20161229-1483015851107029786.jpg)




## lay2 flat 型网络





## lay2 vlan 型网络




## lay2 vxlan 型网络


## lay3 网络



***END***
