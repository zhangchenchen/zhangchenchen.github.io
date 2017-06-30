---
layout: post
title:  "openstack-- neutron 二/三层网络实现探究"
date:   2017-02-12 17:44:25
tags: openstack
---



## 引出

- Neutron 是openstack 中提供网络虚拟化的组件，根据二层网络的实现方式不同(即agent的不同)，可以分为Linux bridge的方式，Openvswitch的方式。而且，lay2 network分为local,flat,vlan,vxlan 等类型(gre与vxlan类似，不再考虑)，本文就分析两种实现方式在这四种网络中的具体实现异同。因为vxlan会依赖lay3层网络，所以还会分析下lay3网络的实现。
- 本文内容主要来自[每天5分钟玩转 OpenStack](http://product.dangdang.com/24160022.html?ref=book-02-L),图片也是来自对应博客，本文仅是对其总结对比并作记录。


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

ovs的实现要相对复杂些，neutron完成相应配置并启动后，ovs-agent会调用ovs自动创建三个ovs bridge，br-ex,br-int和br-tun，看名字大概也可以猜出，br-ex是用于外部网络，br-int是内部虚拟机的网络，br-tun是用于overlay network,也就是vxlan类型的网络会用到该bridge。local network 只需考虑br-int bridge。示例图如下：

![local-by-ovs](http://7xo6kd.com1.z0.glb.clouddn.com/upload-ueditor-image-20161229-1483015852547009927.jpg)

- 图中每个instance并不是跟之前一样通过tap设备直接挂载在ovs的 br-int上，而是通过一个新建的linux bridge以及一对veth pair跟br-int相连，为什么这样做呢，因为Open vSwitch 目前还不支持将 iptables 规则放在与它直接相连的 tap 设备上，这样就实现不了Security group，所以就加了一个linux bridge以支持iptables。
- 两个local network都有自己的DHCP server,由dnsmasq在各自独立的net namespace实现，通过tap设备挂接在各自network的bridge上。
- 都挂载在同一个br-int上，如何区分不同的local network呢，其实，br-int上挂载的虚拟网卡或DHCP对应的port都有一个特殊的tag属性。同一网络的tag相同，不同网络的tag不同，这里跟vlan的vlan ID类似，同一tag属性的port是二层连通的。
![port-tag](http://7xo6kd.com1.z0.glb.clouddn.com/upload-ueditor-image-20161229-1483015851107029786.jpg)




## lay2 flat 型网络


flat型网络是在local 型网络的基础上实现不同宿主机的instance之间二层互联。但是每个flat network都会占用一个宿主机的物理接口，所以生产环境中也不会使用。


### linux bridge 实现flat 型网络

每一个flat network对应一个物理网卡，该对应关系需要在ml2_conf.ini配置文件中指明。下图为flat network示例：

![linux-bridge-flat-net](http://7xrnwq.com1.z0.glb.clouddn.com/linux-bridge-flat-network.png)

- 该 flat network 在计算和控制节点都使用eth1作为接口。如果是新建另一个flat network,只能使用除eth1之外的其他物理接口，且在配置文件中添加相应的mapping配置。
- 同一网络，不同宿主机上的Linux bridge 名称是一样的。
- 只在控制节点有DHCP的设备，因为同一网络只使用一个DHCP server。
- 其他跟local network类似。

### openvswitch 实现flat 型网络

同linux bridge实现的flat network一样，每一个flat network会占用一个物理接口，需要在配置文件ml2_conf.ini中指定对应关系，这里，需要利用ovs新建一个ovs bridge，将该bridge挂载对应物理接口。下图为flat network 示例：

![ovs-flat-net](http://7xo6kd.com1.z0.glb.clouddn.com/upload-ueditor-image-20170108-1483848357938049015.jpg)

- br-eth1就是我们新建的ovs bridge,为什么要新建一个ovs bridge与物理网卡连接，而不是直接用br-int与物理网卡连接呢，因为我们在配置文件中需要配置的是某一flat network 的label(flat network type,就是一个标识)与ovs-bridge的mapping,如果是直接用br-int与物理网卡连接的话，那么当建立多个flat-network的时候，br-int与多个物理网卡相连，br-int是无法辨识的。而linux bridge实现flat network的时候是不同的linux bridge与不同的物理网卡连接。
- 当再新建一个flat network 的时候需要再新建一个ovs bridge，连接另一网卡， 并于配置文件指定mapping。
- 这里出现了一种新型的网络设备叫做patch port,就是br-eth1与br-int相连的设备，与veth pair的功能类似，只不过这种设备只适用于两个ovs bridge之间互联。
- 只在控制节点有DHCP的设备，因为同一网络只使用一个DHCP server。
- br-int内不同网络的区分跟local network一样，也是通过tag实现。

## lay2 vlan 型网络

### linux bridge 实现vlan 型网络

vlan network是在flat network的基础上实现多个不同的vlan network 共用同一个物理接口。需要在配置文件中指定vlan的范围（租户的网络id 范围），以及 vlan network 与物理网卡的对应关系。
vlan network如下示例：

![vlan-linux-bridge](http://7xrnwq.com1.z0.glb.clouddn.com/vlan-linux-bridge.png)

- 可以看出，bridge上除了挂载instance的tap,dhcp的tap设备外，还挂载了一个vlan interface eth1.10x ,该vlan interface 便是区分不同vlan的关键。每一个vlan network有一个对应的vlan id，该vlan id 对应相应数字的vlan interface。
- 上图所示，有两个vlan,vlan 100 和vlan 101,vm1 与 vm2 在同一网络vlan100，是可以联通的，vm3 与vm1,vm2不连通。


### openvswitch 实现vlan 型网络


- 配置文件指定tenant网络类型,以及vlan id范围，对应的ovs bridge mapping。
- 与 Linux Bridge driver 不同，Open vSwitch driver 并不通过 eth1.100, eth1.101 等 VLAN interface 来隔离不同的 VLAN。所有的 instance 都连接到同一个网桥 br-int。Open vSwitch 通过 flow rule（流规则）来指定如何对进出 br-int的数据进行转发，进而实现 vlan 之间的隔离。
vlan network如下示例：

![vlan-ovs-network](http://7xo6kd.com1.z0.glb.clouddn.com/upload-ueditor-image-20170119-1484822850549065346.jpg)

- 乍一看，跟ovs 实现的flat network差不多，不过该图示有两个vlan network，都通过br-eth1与外界宿主机的instance联通。
- 下面看下Open vSwitch 如何通过 flow rule（流规则）进行vlan 的隔离。通过 ovs-ofctl dump-flows $BRIDGE 命令查看指定bridge 的flow rule。
![flow-rule](http://7xo6kd.com1.z0.glb.clouddn.com/upload-ueditor-image-20170119-1484822850816040765.jpg)
简单解释下：每一行代表一条rule,priority代表该rule的优先级，值越大优先级越高，in_port代表传入数据的port编号，每个port在ovs bridge中 都会有一个编号，可以通过 ovs-ofctl show $BRIDGE 命令查看 port 编号。
![port-num](http://7xo6kd.com1.z0.glb.clouddn.com/upload-ueditor-image-20170119-1484822851415080454.jpg)
如上图：eth1 编号为 1；phy-br-eth1 编号为 2。 
dl_vlan是数据包原始的 VLAN ID，actions表示对数据包进行的操作。
那么第一条rule表示的意思就是：从 br-eth1 的端口 phy-br-eth1（in_port=2）接收进来的包，如果 VLAN ID 是 1（dl_vlan=1），那么需要将 VLAN ID 改为 100（actions=mod_vlan_vid:100。为什么要转换呢，之前讲local/flat network时讲过， br-int为了区分不同网络会给挂载的设备打上tag,这里的tag就类似vlan id，不过是仅限于br-int可以识别，宿主机之外是无法识别的，所以需要将该tag转换为外界可以识别的vlan id。
再看下br-int 的flow rule:
![br-int-flow-rule](http://7xo6kd.com1.z0.glb.clouddn.com/upload-ueditor-image-20170119-1484822851781065438.jpg)

图中圈出来的第一条rule的意思就是：从int-br-eth1接收进来的数据包，如果 VLAN 为 100，则改为内部 VLAN 1。
其他rule分析类似。




## lay3 网络

lay3 网络主要是一个路由的功能，即实现两个不同subnet之间instance的互联。除此之外，与外网的互联，firewall以及instance attach floating-ip 等功能也都是由L3 agent实现的虚拟路由器（net namespace + iptables）完成的。


### linux bridge 实现lay3 网络

首先需要修改配置文件，启用 l3 agent,配置文件位于控制节点/网络节点的/etc/neutron/l3_agent.ini，mechanism driver 是linux bridge(或openvswitch)。
创建一个router后，如下示例：

![router-ex](http://7xo6kd.com1.z0.glb.clouddn.com/upload-ueditor-image-20161018-1476795972212028903.jpg)

看一下router是如何连接两个vlan 的subnet: l3 agent 会为每个 router 创建了一个 namespace，通过 veth pair 与 TAP 相连，然后将 Gateway IP 配置在位于 namespace 里面的 veth interface 上，这样就能提供路由了。
![router-namespace](http://7xo6kd.com1.z0.glb.clouddn.com/upload-ueditor-image-20161018-1476795971897066009.jpg)
这样便实现 vlan100 和 vlan 101 的连通了。

再简单说下flaoting-ip attach的实现原理，floating-ip 的出现是为了让外网能够直接访问到被attached的instance。
![floating-ip](http://7xo6kd.com1.z0.glb.clouddn.com/upload-ueditor-image-20161027-1477564018661061183.jpg)
如图，是有两个内部vlan network和一个external network（网络类型可能是flat或vlan）,一个虚拟路由器将这三个网络连在一起，vm1通过虚拟路由器可以访问外网，但是外网没法直接主动连接vm1.我们将一个flaoting-ip 如10.10.10.3，attach到vm1之后，发现外网可以通过该floating-ip 直接连接vm1.进入vm1，看下它的网卡配置，发现并没有floating-ip对应的网卡存在。怎么回事呢，其实真正实现floating-ip attach操作的是发生在虚拟路由器上，查看下对应的虚拟路由器的网卡，
![router-interface](http://7xo6kd.com1.z0.glb.clouddn.com/upload-ueditor-image-20161101-1478002791162005280.jpg?_=6020949)

再看下虚拟路由器iptables的nat配置，
![router-iptables](http://7xo6kd.com1.z0.glb.clouddn.com/upload-ueditor-image-20161101-1478002791526056451.jpg?_=6020949)

这样，便明白了，floating IP 是配置在 router 的外网 interface 上的，而非 instance，floating IP 能够让外网直接访问租户网络中的 instance，这是通过在 router 上应用 iptalbes 的 NAT 规则实现的。 


### openvswitch 实现lay3 网络

与linux bridge 的实现类似，只不过这里不是挂载在不同bridge上，而是直接挂载在br-int上。

![ovs-layer3-net](http://7xo6kd.com1.z0.glb.clouddn.com/upload-ueditor-image-20170124-1485241324112000116.jpg)

两个 Gateway IP 分别配置在 qr-2ffdb861-73 和 qr-d295b258-45 上,从而实现两个subnet的连接。
floating-ip attach 与Linux bridge类似实现。

## lay2 vxlan 型网络

除了local, flat, vlan 这几类网络，OpenStack 还支持 vxlan 和 gre 这两种 overlay network。所谓overlay，是指指建立在其他网络上的网络，比如vxlan就是在udp的基础上实现。 vxlan 和 gre 都是基于隧道技术实现的。目前 linux bridge 只支持 vxlan，不支持 gre；open vswitch 两者都支持。因为vxlan与gre类似，这里只讨论vxlan。
VXLAN 提供与 VLAN 相同的以太网二层服务，但是拥有更强的扩展性和灵活性。与 VLAN 相比，VXLAN 有下面几个优势：

1. 支持更多的二层网段。 VLAN 使用 12-bit 标记 VLAN ID，最多支持 4094 个 VLAN，这对于大型云部署会成为瓶颈。VXLAN 的 ID （VNI 或者 VNID）则用 24-bit 标记，支持 16777216 个二层网段。
2. 能更好地利用已有的网络路径。 VLAN 使用 Spanning Tree Protocol 避免环路，这会导致有一半的网络路径被 block 掉。VXLAN 的数据包是封装到 UDP 通过三层传输和转发的，可以使用所有的路径。
3. 避免物理交换机 MAC 表耗尽。 由于采用隧道机制，TOR (Top on Rack) 交换机无需在 MAC 表中记录虚拟机的信息。


VXLAN 是将二层建立在三层上的网络。 通过将二层数据封装到 UDP 的方式来扩展数据中心的二层网段数量。 VXLAN 是一种在现有物理网络设施中支持大规模多租户网络环境的解决方案。 VXLAN 的传输协议是 IP + UDP。

VXLAN 定义了一个 MAC-in-UDP 的封装格式。 在原始的 Layer 2 网络包前加上 VXLAN header，然后放到 UDP 和 IP 包中。 通过 MAC-in-UDP 封装，VXLAN 能够在 Layer 3 网络上建立起了一条 Layer 2 的隧道。

关于 vxlan 的包格式以及相关的设备的，参考[VXLAN 概念（Part I） - 每天5分钟玩转 OpenStack（108）](https://www.ibm.com/developerworks/community/blogs/132cfa78-44b0-4376-85d0-d3096cd30d3f/entry/VXLAN_%E6%A6%82%E5%BF%B5_Part_I_%E6%AF%8F%E5%A4%A95%E5%88%86%E9%92%9F%E7%8E%A9%E8%BD%AC_OpenStack_108?lang=en)


### linux bridge 实现vxlan 网络

[VXLAN 概念（Part II）- 每天5分钟玩转 OpenStack（109）](https://www.ibm.com/developerworks/community/blogs/132cfa78-44b0-4376-85d0-d3096cd30d3f/entry/VXLAN_%E6%A6%82%E5%BF%B5_Part_II_%E6%AF%8F%E5%A4%A95%E5%88%86%E9%92%9F%E7%8E%A9%E8%BD%AC_OpenStack_109?lang=en)

### ovs 实现vxlan 网络

[OVS vxlan 底层结构分析 - 每天5分钟玩转 OpenStack（148）](http://www.cnblogs.com/CloudMan6/p/6376523.html)


***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***
