---
layout: post
title:  "openstack-- Open vSwitch 安装以及常用操作"
date:   2017-03-16 14:10:32
tags: 
  - openstack
  - openvswitch
---



## 引出
openvswitch,简称ovs, 是一个开源的分布式虚拟多层交换机，随着云计算的崛起，它逐渐成为网络虚拟化的基石。在openstack 中，默认也是采用ovs作为二层网络实现的组件。本篇博客即是对ovs安装，以及框架，常用命令的一个记录。


## 在CentOS 7 上安裝 Open vSwitch

### 确定linux核心版本,以安装对应版本OVS

```bash
$ uname -r
```
根据下图查看需要安装哪一个版本，一般来说，如果kernel支持的话，最好下载2.5.1版，也就是目前的LTS(Long Term Support)版。
![ovs-kernal](http://7xrnwq.com1.z0.glb.clouddn.com/2017-03-16-ovs-kernal.png)

### 安装 toolchain 以及一些依赖包

目前ovs安装还是需要自己编译，所以需安装toolchain。

```bash
$ yum groupinstall "Development Tools"

$ yum install openssl-devel wget kernel-devel
```


### 下载源文件并打包为rpm包
```bash
$ wget http://openvswitch.org/releases/openvswitch-2.5.1.tar.gz  #下载

$ tar xzvf openvswitch-2.5.1.tar.gz  #解压缩

$ mkdir -p ~/rpmbuild/SOURCES & cp openvswitch-2.5.1.tar.gz ~/rpmbuild/SOURCES/  #创建一个打包目录并复制源文件

$ sed 's/openvswitch-kmod, //g' openvswitch-2.5.1/rhel/openvswitch.spec > openvswitch-2.5.1/rhel/openvswitch_no_kmod.spec # 修改spec文件

$ rpmbuild -bb --nocheck ~/openvswitch-2.5.1/rhel/openvswitch_no_kmod.spec # 打包
```

完成后我们在~/mbuild/RPMS/x86_64/目录下有两个rpm包，其中，openvswitch-2.5.1-1.x86_64.rpm就是我们需要的rpm包。


### 安装rpm包

```bash
$  yum localinstall /root/rpmbuild/RPMS/x86_64/openvswitch-2.5.1-1.x86_64.rpm

$ ovs-vsctl -V #查安装是否成功

$ systemctl start openvswitch.service #启动ovs service（即守护进程）

$ chkconfig openvswitch on #设置开机自启动

```



## OpenvSwitch的架构以及基本概念


### ovs架构

![ovs-arc](http://oeptotikb.bkt.clouddn.com/2017-03-11-openvswitch-arch.png)

主要由三个组件组成：

- ovs-vswitchd ：ovs的守护进程。
- ovsdb-server ：ovs的轻量数据库服务。
- openvswitch_mod.ko ：ovs的内核模块，主要是利用datapath将flow的match结果缓存起来。


### ovs 基本概念

因为ovs最常用到的场景是neutron，所以这里以neutron vxlan为例子讲下，下图就是一个neutron vxlan的示意图：

![neutron-vxlan](http://7xrnwq.com1.z0.glb.clouddn.com/2017-03-16-neutron-vxlan.png)


 - bridge: ovs中的网桥可以理解为现实世界中的物理交换机，作用就是根据一定流规则（如openflow），把从某个端口(port)收到的数据包转发到另一个或多个端口。图中br-int与br-tun就是两个bridge。
 - port: ovs 中的port是收发数据包的单元,每个端口都附属于某一个bridge。图中qvoxxx，以及patch-tun,patch-int等都是port。
 - interface: 接口是ovs与外部交换数据包的组件。一个接口就是操作系统的一块网卡，这块网卡可能是ovs生成的虚拟网卡，也可能是物理网卡挂载在ovs上，也可能是操作系统的虚拟网卡（TUN/TAP）挂载在ovs上。
 - flowtable: 上文讲过，ovs会根据流表进行包的转发，最常用的就是openflow协议，下图为openflow的流表匹配顺序：首先按照从小到大的顺序匹配流表，表内按照优先级匹配表项，示例参见[openstack-- neutron 二/三层网络实现探究](https://zhangchenchen.github.io/2017/02/12/neutron-layer2-3-realization-discovry/)。

  ![openflow-match](http://7xrnwq.com1.z0.glb.clouddn.com/2017-03-16-openvswitch-openflow-match.png)

## OpenvSwitch的常用命令

ovs的操作命令大致有如下四种：

- ovs-vsctl用于控制ovs db
- ovs-ofctl用于管理OpenFlow switch 的 flow
- ovs-dpctl用于管理ovs的datapath
- ovs-appctl用于查询和管理ovs daemon

ovs的常用子命令如下,直接引用自[OVS常用操作总结](http://fishcried.com/2016-02-09/openvswitch-ops-guide/)：

- ovs-dpctl show -s
- ovs-ofctl show, dump-ports, dump-flows, add-flow, mod-flows, del-flows
- ovsdb-tools show-log -m
- ovs-vsctl
    - show 显示数据库内容
    - 关于桥的操作 add-br, list-br, del-br, br-exists.
    - 关于port的操作 list-ports, add-port, del-port, add-bond, port-to-br.
    - 关于interface的操作 list-ifaces, iface-to-br
    - ovs-vsctl list/set/get/add/remove/clear/destroy table record column [value], 常见的表有bridge, controller,interface,mirror,netflow,open_vswitch,port,qos,queue,ssl,sflow.
- ovs-appctl list-commands, fdb/show, qos/show



## OpenvSwitch实践

参考[基于 Open vSwitch 的 OpenFlow 实践](http://www.ibm.com/developerworks/cn/cloud/library/1401_zhaoyi_openswitch/index.html)



## 参考文章


[CentOS 7 安裝 Open vSwitch](http://qbsuranalang.blogspot.com/2016/11/centos-7-open-vswitch.html)

[Open vSwitch and OpenStack Neutron troubleshooting](http://www.yet.org/2014/09/openvswitch-troubleshooting/)

[ovs wiki](https://en.wikipedia.org/wiki/Open_vSwitch)

[Open vSwitch的ovs-vsctl命令详解](http://www.rendoumi.com/open-vswitchde-ovs-vsctlming-ling-xiang-jie/)

[OVS常用操作总结](http://fishcried.com/2016-02-09/openvswitch-ops-guide/)

***END***
