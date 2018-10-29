---
layout: post
title:  "openstack-- neutron 分布式虚拟路由（DVR）"
date:   2017-06-20 14:30:32
tags: 
  - openstack
  - neutron
---



## 引出

公司有个测试云平台的虚拟机密码初始化出现了问题，根据之前的经验可能跟[这篇](https://zhangchenchen.github.io/2017/02/17/openstack-instance-metadata-discovery/)问题一样,没想到花费了将近一天的时间才将问题解决，发现最终是跟neutron 分布式虚拟路由有关，这里记录一下。

问题是这样的：nova boot的时候传入 user-data（user-data中含有需要修改的密码），instance起来的时候会执行cloud-init，然后会使用EC2的datasource方式（即“169.254.169.254”）去获取user-data，然而发现密码没有更改，查看日志发现,发现获取user-data时，报错如下

  >Calling 'http://169.254.169.254/2009-04-04/meta-data/instance-id' failed [0/120s]: bad status code [500]

初步断定就是instance metadata 的某个环节网络不通。

## 问题发现

- 第一步想到的就是重启下相关服务，将neutron-metada-agent,neutron-l3-agent,nova-api重启后发现问题依旧。
- 查看相关日志，没有找到有关报错信息，而且奇怪的是网络节点对应的neutron-ns-metadata-proxy-xxxxxxxxx.log都没有处理请求的日志，说明请求可能没有走到这里。
- 只能按照请求的处理步骤一步步排查，首先确定虚拟机所在的网络是有路由器相连的，所以metadata请求是走的路由器的namespace，相关知识参考[openstack-- openstack instance metadata 服务机制探索](https://zhangchenchen.github.io/2017/02/17/openstack-instance-metadata-discovery/),网络节点对应路由器的namespace中一切正常，9697 端口的重定向，以及监听9697端口的neutron-metadata-ns-proxy进程也正常。没办法，在该路由器的namespace中对应端口抓包试试，虚拟机中发送169.254.169.254，却一点反应没有，断定请求压根就没有走到该路由器的namespace。只能返回去虚拟机实例中抓包试试，使用traceroute命令发现请求是发到对应网关的，而对应网关就应该在路由器的namespace中啊，到这里基本就蒙逼了。
- 在经过一番徒劳无功的试错后，终于找到了一点线索。我在查看neutron agent状态的时候，发现neutron-l3-agent不只是在网络节点，计算节点也存在neutron-l3-agent，立马想到neutron-l3-agent会在本地计算节点也创建虚拟路由器，所以获取metadata的请求就发送到本地计算节点的路由器 namespace，果然，本地计算节点也有一个路由器，抓包发现确实是发送到了这里，而且该节点对应网络的neutron-ns-metadata-proxy-xxxxxxxxx.log发现了 socket.error报错信息，说明neutron-ns-metadata-proxy通过 unix domian socket 转发给 neutron-metadata-agent的时候出现了问题。本能的想是不是配置错误导致的，查看l3-agent的配置文件时发现有个配置项：agent_mode = dvr ，赶紧搜了一下，终于找到了进入的正确姿势。



## neutron 分布式虚拟路由（DVR）


### 整体架构 

neutron 的DVR 是属于openstack HA的一部分，主要解决虚拟路由器的HA（即neutron-l3-agent的HA）。之前写过一篇[记录openstack的HA](https://zhangchenchen.github.io/2017/04/14/openstack-ha/),但没有仔细研究过neutron DVR。

在没有使用dvr时，我们的整体架构大致是这样的：

![neutron-arc1](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/neutron-arc-1.png)

计算节点只需要安装L2 Agent就可以，而使用dvr模式后：

![neutron-arc-2](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/2017-06-20neutro-arc-2.png)

计算节点除了L2 Agent外，还要安装L3 Agent以及Metadata Agent。也就是说，通过使用dvr，计算节点也有了网络节点的三层转发和NAT功能，起到了分流的作用。

### 安装配置

- 控制节点：修改neutron.conf 如下，重启neutron-server

```yaml
[DEFAULT]
router_distributed = True
```

- 网络节点：修改 openswitch_agent.ini如下，重启Open vSwitch agent

```yaml
[DEFAULT]
enable_distributed_routing = True
```

修改l3_agent.ini如下，重启Layer-3 agent，

```yaml
[DEFAULT]
agent_mode = dvr_snat
```

- 计算节点：安装l3-agent,metadata-agent,修改 openswitch_agent.ini如下，重启Open vSwitch agent

```yaml
[DEFAULT]
enable_distributed_routing = True
```
修改l3_agent.ini 如下，重启l3-agent

```yaml
[DEFAULT]
interface_driver = openvswitch
external_network_bridge =
agent_mode = dvr
```

可以通过 neutron agent-list 验证agent 是否启动。

### 网络流通方向

![deploy-ovs-ha-dvr-compconn](https://docs.openstack.org/newton/networking-guide/_images/deploy-ovs-ha-dvr-compconn1.png)

照着这张图梳理一遍以下几种情况的网络走向：

1. 没有floating-ip的instance的南北向网络流通情况
  即图中蓝色标注的线路，以图中左边计算节点的instance为例，首先通过一对veth设备与一个linux-bridge相连（该linux-bridge的存在主要是为了利用iptables实现Sec-group），又是一对veth设备，连接到OVS生成的br-int网桥，该instance在三层网络的第一跳便是到与该br-int相连的Disk router namespace的网关接口，因为没有floating-ip ，所以不会通过FIP namespace，而是通过patch设备到br-tun，再到网络节点的br-int,到达snat namespace，进行snat(源地址转换)，最后通过ovs-provider-bridge与外网联通。

2. 有floating-ip的instance的南北向网络流通情况
  刚开始与第一种情况类似，但是跳到本地router namespace后，接下来会跳到本地的RIP namespace，做snat,然后直接通过本地的ovs-provider-bridge连接到外网。 
3. 不同子网但有虚拟路由连接的instance东西向网络
  刚开始类似，之后在本地router namespace进行路由选择，并通过br-int,br-tun进入对应虚拟机的计算节点（该部分工作由ovs 的openflow完成，同时还完成了snat），到了 目标计算节点上，依次被 br-tun，br-int 处理，直到通过 tap 设备进入另一instance。

关于详细描述以及实验可以参考[官方文档](https://docs.openstack.org/newton/networking-guide/deploy-ovs-ha-dvr.html#deploy-ovs-ha-dvr),以及[这篇](http://www.cnblogs.com/sammyliu/p/4713562.html)。


## 问题解决

回到刚开始碰到的问题，确定了问题的所在，就是在本地计算节点neutron-ns-metadata-proxy通过 unix domian socket 转发给 neutron-metadata-agent的时候出现了问题，neutron-ns-metadata-proxy是正常的，那么问题就出在neutron-metadata-agent上，果然，该agent在计算节点并没有启动，启动后正常。



## 参考文章


[Network Troubleshooting](https://docs.openstack.org/ops-guide/ops-network-troubleshooting.html)

[Neutron Networking: Neutron Routers and the L3 Agent](https://developer.rackspace.com/blog/neutron-networking-l3-agent/)

[Neutron/DVR](https://wiki.openstack.org/wiki/Neutron/DVR)

[理解 OpenStack 高可用（HA）（3）：Neutron 分布式虚拟路由（Neutron Distributed Virtual Routing）](http://www.cnblogs.com/sammyliu/p/4713562.html)

[Neutron 理解 (6): Neutron 是怎么实现虚拟三层网络的 [How Neutron implements virtual L3 network]](http://www.cnblogs.com/sammyliu/p/4636091.html)

[Neutron DVR实现multi-host特性打通东西南北流量提前看](http://blog.csdn.net/quqi99/article/details/20711303)

[Open vSwitch: High availability using DVR](https://docs.openstack.org/newton/networking-guide/deploy-ovs-ha-dvr.html#deploy-ovs-ha-dvr)

***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***
