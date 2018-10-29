---
layout: post
title:  "openstack-- 对neutron一些常见网络问题的troubleshooting"
date:   2017-03-13 11:10:32
tags: 
  - openstack
  - neutron
---



## 引出

一直对neutron网络的troubleshooting 感到头疼，今天偶然翻到一篇博客，感觉写的非常棒，[Openstack Neutron: troubleshooting and solving common problems](http://abregman.com/2016/01/06/openstack-neutron-troubleshooting-and-solving-common-problems/),这篇博客也是两个教学视频的总结，顺便看了一下视频，感觉内容差不多就没看完，链接在后面。

本篇博客算是对上述博客以及视频的一个翻译（没有严格翻译）与整理，以及自己补充的一些东西。主要解决四类常见的网络问题：
- 无法用private ip  ping/ssh 通虚拟机
- 虚拟机无法连通外部网络
- 虚拟机无法连通 metadata server
- VIF plugging timeout (vif 插件超时)


## 常见问题分类

问题的原因通常来说分为以下两种：

- 配置错误
这是最常见的错误原因，因为配置文件的错误配置导致失败。这里的配置文件不止是neutron的配置文件，还有相关的nova,cinder,各种agent甚至物理机网络的配置。需要注意的是，宿主机网络的配置错误会严重影响neutron 的使用，因为所有的网络包最终都是通过物理网络来传输。所以，对neutron 进行trouble shooting之前，首先确定宿主机之间是可以联通的。

- 代码出现bug
出现这种情况的话，一般在neutron的[launchpad](https://bugs.launchpad.net/neutron/) 会有对应的bug report。如果没有的话，那就自己提一个，会有社区的developer进行 bug fixed。


## 第一类问题：无法用private  ip  ping/ssh 通虚拟机

在解决这个问题之前，我们首先要了解一个虚拟机是如何获取IP的。

在neutron 中，有一个agent叫做 DHCP agent，它通过 RPC 跟 neutron-server 通信，它会为每一个 network 创建一个dhcp namaspace，以实现网络的隔离。在这个namespace里，有一个dnsmasq的进程，它是真正实现分配IP 的服务。

![vm-get-ip](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/2017-03-13-vm_get_ip.png)

关于这部分的具体代码分析，可以参考[Neutron 理解（5）：Neutron 是如何向 Nova 虚机分配固定IP地址的 （How Neutron Allocates Fixed IPs to Nova Instance）](http://www.cnblogs.com/sammyliu/p/4419195.html)
接下来我们看下packets 的flow流向，以便更精准的定位问题,因为二层网络的实现方式不一样——————openvswitch方式和Linux bridge方式，所以流向也不尽相同，先看下ovs的流向。

![ovs-get-ip](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/2017-03-13-openvswitch_flow.png)

流程图一目了然，原文其实做了很多解释，这里就不多说了，如果不清楚的可以参看原文，或者看我之前的一篇[openstack-- neutron 二/三层网络实现探究](https://zhangchenchen.github.io/2017/02/12/neutron-layer2-3-realization-discovry/)

同样，linux bridge实现的二层网络，流向如下：

![bridge-get-ip](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/2017-03-13-linux_bridge_flow.png)

ok,接下来开始debug:

- 首先利用nova list 确定虚拟机是 active状态,如果虚拟机是failure，那么可以用 nova show 查看failure原因，以及查看 nova日志。
- 确定虚拟机是running 的，接下来就要进行网络的分析，首先需要确定两件事：一是物理网络是没问题的，二是虚拟机的security group是允许ping/ssh的。
- 接下来查看port绑定是否成功，分为在虚拟机上的port绑定和在DHCP/router上的绑定。一般来说虚拟机上的port绑定是比较容易被发现的，因为在boot虚拟机的过程中一旦绑定不成功就会报错。而在DHCP/router上的绑定，因为是异步过程所以不容易被发现（当创建一个port的时候即使最终port binding失败，但是也会返回创建成功）。这就需要我们使用 neutron port-show  PORT_ID 查看对应port是否绑定成功，不成功的话会如下图所示：

![bonding-failure](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/2017-03-13-binding_failure.png)

   一般来说，port绑定不成功的话有两个原因：第一个原因是OVS agent 挂掉了，可以使用 neutron agent-list查看，OVS agent挂掉的另一个显著特征是 ovs-vsctl show | grep tap -A 3 发现br-int网桥的tap设备没有vlan tag. 第二个原因就是neutron agent 或者neutron server配置错误。

- 接下来看一下虚拟机是否获取到IP。通过 vnc-console 进入虚拟机，执行ip a 命令，如果没有ip的话，首先查看neutron dhcp agent是否挂掉，还是用neutron agent-list命令查看。如果agent没有挂掉的话，再去查看下对应network的dnsmasq服务是否挂掉，使用该命令： ps -ef | grep dnsmasq | grep Network_Id 。再去查看一下对应network的dhcp host文件，是否有对应虚拟机的mac与ip 记录，该文件的位置为：/var/lib/neutron/dhcp/Network_ID/host。如果还是一切正常，那么去查看dhcp-agent 的日志。同时，我们要保证宿主机与虚拟机之间网络是可以联通的，可以通过手动设置虚拟机ip的方法进行验证下。
- 如果还是没有解决问题，那就只有一个方法了，使用tcpdump 抓包吧。



## 第二类问题：虚拟机无法连通外部网络

解决这类问题，首先要弄懂L3 agent的工作原理。说白了，就是利用namespace+iptables实现的，参考[openstack-- neutron 二/三层网络实现探究](https://zhangchenchen.github.io/2017/02/12/neutron-layer2-3-realization-discovry/)。

还是看下包流向，因为2层网络的实现方式不同，流向也不尽相同。首先看下ovs方式的流向：

![ovs-flow](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/2017-03-13-l3_external_flow.png)

再看下linux bridge的方式，图示为从外部网络ping 虚拟机floating ip过程。

![bridge-extenal-flow](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/2017-03-13-l3_external_linux_bridge_flow.png)

debug步骤如下：

- 查看security group 是否允许ping。可以的话，首先要保证宿主机与private ip之间是可以ping通的，如果两者ping不通，那么使用floating ip也是绝壁不可能ping通的。
- 接下来从两个角度入手：从虚拟机到网络节点的对应router是否联通，从router到external net是否联通。
- 从虚拟机到router:通过 vnc-console进入虚拟机，查看ip a 是否获取到ip,查看路由信息：route -n，查看能否ping 通默认网关（即路由器IP）
- 从router到external net：查看在router namespace内能否ping通外部网络：ip netns exec qrouter-078766fe-c4b6-4a14-82fa-e7f85e26c248 ping www.baidu.com   。查看router namespace能否ping通 floating ip: ip netns exec qrouter-078766fe-c4b6-4a14-82fa-e7f85e26c248 ping Floating-IP .这看起来可能有点傻，因为floating ip其实就在这个namespace中，不过这会让我们对问题的严重性有一个认识。
- 如果还是没有找到问题，就去查看l3-agent 日志。



## 第三类问题：虚拟机无法连通 metadata server

还是先要搞明白metadata server的原理。metadata 获取有两种方式，一种是config drive，一种是metadata server。这里就是讨论metadata server的方式，metadata server 的路由方式有两种：如果虚拟机与router联通的话，就通过router路由，如果虚拟机没有router直连的话，就通过对应dhcp namespace路由。参考[openstack-- openstack instance metadata 服务机制探索](https://zhangchenchen.github.io/2017/02/17/openstack-instance-metadata-discovery/)。

首先看下通过路由器路由的work-flow:

![router-work-flow](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/2017-03-13-routed_networks_metadata.png)

注意：metdata proxy 是由l3-agent实现，它会在指定端口监听（默认 9697），接收到request后，会增加虚拟机ip,router id到request header并转发到metadata agent。

再看下没有路由器直连（isolated-network）的情况:

![isolated-net](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/2017-03-13isolated_network_metadata.png)

注意：要想开启这一功能，需要在dhcp-agent配置文件中设置：   
enable_isolated_metadata = True.

接下来开始debug:

- 首先查看metadata-agent是否挂掉：neutron agent-list
- 还是两个角度：从虚拟机到router/dhcp namespace，从router/dhcp namespace 到nova-api-metadata。
- 从虚拟机到router/dhcp namespace：ping一下查看是否连通。
- 查看router/dhcp namespace 是否有metadata-proxy 在监听：ip netns exec qrouter-f6396cfe-2ac9-4d6a-9437-eb8e7d26c776 ps -ef | grep metadata-proxy 。如果这里出现错误的话，可以去查看下metadata-agent 的日志，以及对应 namespace的neutron-ns-metadata-proxy的日志。
- 从router/dhcp namespace 到nova-api-metadata：ip netns exec qrouter-f6396cfe-2ac9-4d6a-9437-eb8e7d26c776 ping Metadata-Server IP
- 终极大招：tcpdump



## 第四类问题： VIF plugging timeout

这类问题通常出现在boot虚拟机的时候，跟L2 agent(ovs/bridge agent)有关。

workflow如下：

![vif-plugging](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/2017-03-13vif_plugging.png)

当nova 发出allocate_network 请求时，会设置一个默认5分钟的等待时间，如果5分钟过后还没有收到neutron 回复，就会报 VIF plugging timeout 这个错误。

debug步骤：

- 查看对应host的nova-computer日志，以及openvswitch-agent日志。
- 查看网络节点neutron-server日志。
- 如果是做压测，在Nova的配置文件中，可以适当调大这个vif_plugging_timeout 时间，或增加rpc_thread_pool_size & rpc_conn_pool_size 。




原文还列举了一些常用的网络调试命令，因为之前总结过，[openstack--openstack 常用网络调试命令](https://zhangchenchen.github.io/2017/02/10/openstack-common-net-command/),不再赘述。





## 参考文章


[Openstack Neutron: troubleshooting and solving common problems](http://abregman.com/2016/01/06/openstack-neutron-troubleshooting-and-solving-common-problems/)

[I Can't Ping My VM! Learn How to Debug Neutron and Solve Common Problems](https://www.youtube.com/watch?v=aNA8Pvewu2M)

[ OpenStack Neutron Troubleshooting ](https://www.youtube.com/watch?v=O9pghlsNG9E)

[Neutron 理解（5）：Neutron 是如何向 Nova 虚机分配固定IP地址的 （How Neutron Allocates Fixed IPs to Nova Instance）](http://www.cnblogs.com/sammyliu/p/4419195.html)

***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***
