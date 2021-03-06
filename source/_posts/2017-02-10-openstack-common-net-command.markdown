---
layout: post
title:  "openstack--openstack 常用网络调试命令"
date:   2017-02-10 14:54:25
tags: openstack
---



## 引出

openstack网络部分应该算是最为复杂的，这里简单罗列下常用的命令作为备注，主要包括以下三个部分：

1. neutron 命令
2. ip netns 命令（即网络命名空间）
3. ovs/brctl 命令（即ovs和网桥命令）
4. iptables 命令


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



## ip netns 命令

- 列出宿主机所有网络命名空间：ip netns
- 在某一网络命名空间执行命令：ip netns exec $NAMESPACE COMMAND  ,示例：
        1. ip netns  exec qrouter-a123f550-747b-4b67-8006-2c29a5dd4c6d ip a
        2. ip netns  exec qrouter-a123f550-747b-4b67-8006-2c29a5dd4c6d ping www.baidu.com 


## ovs/brctl 命令

主要是针对 openvswitch 创造的网桥信息 和 linux bridge 创造的网桥信息

1. 显示ovs创建的网桥信息：ovs-vsctl show  
2. 查看网桥br-tun上的流表信息：ovs-ofctl dump-flows br-tun
3. ovs 创建网桥br-eth0：ovs-vsctl add-br br-eth0
4. 将网卡eth0 桥接在br-eth0上：ovs-vsctl add-port br-eth0 eth0
5. 查看datapath统计信息：ovs-dpctl show -s
6. 显示linux bridge 创建的网桥信息：brctl show 
7. linux bridge 创建网桥br-eth1: brctl addbr br-eth1
8. 将网卡eth1 桥接在br-eth1上: brctl addif br-eth1 eth1


## iptables 命令

neutron 中的L3-agent 默认是用iptables来实现3层网络的实现，而且，security-group, Firewall的实现也都与iptables息息相关。

1. 新增规则到某个规则链的最后一个：iptables -A INPUT ...  
2. 删除某个规则：iptables -D INPUT --dport 80 -j DROP 
3. 取代现行规则，顺序不变：iptables -R INPUT 1 -s 192.168.0.1 -j DROP  
4. 插入一条规则：iptables -I INPUT 1 --dport 80 -j ACCEPT  
5. 列出某规则链中的所有规则：iptables -L INPUT  
6. 列出nat表所有链中的所有规则：iptables -t nat -L  


## 参考文章

[Linux 防火墙和 iptables](http://liaoph.com/iptables/)

[Linux iptables 命令行操作常用指令](https://cnzhx.net/blog/common-iptables-cli/)




***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***
