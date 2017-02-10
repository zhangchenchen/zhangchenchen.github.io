---
layout: post
title:  "openstack--openstack 常用网络调试命令"
date:   2017-02-10 14:54:25
tags: openstack
---



<a name="A"></a>

## 引出

openstack网络部分应该算是最为复杂的，这里简单罗列下常用的命令作为备注，主要包括以下三个部分：

1. neutron 命令
2. ip netns 命令（即网络命名空间）
3. ovs/brctl 命令（即ovs和网桥命令）




<a name="B"></a>

## neutron 命令

- 对net, subnet, port 这些core service的操作，略
- 对ext service 的操作(略)，常见ext service 包括：
        1. router
        2. firewall 
        3. loadbalance
        4. security-group
        5. vpn
        6. floatingip 
- 查看agent列表：neutron agent-list
- 查看service provider: neutron service-provider-list


<a name="B"></a>

## ip netns 命令

- 列出宿主机所有网络命名空间：ip netns
- 在某一网络命名空间执行命令：ip netns exec $NAMESPACE COMMAND  ,示例：
        1. ip netns  exec qrouter-a123f550-747b-4b67-8006-2c29a5dd4c6d ip a
        2. ip netns  exec qrouter-a123f550-747b-4b67-8006-2c29a5dd4c6d ping www.baidu.com 



## ovs/brctl 命令

主要是查看 openvswitch 创造的网桥信息 和 linux bridge 创造的网桥信息
- ovs-vsctl show
- brctl show 

***END***
