---
layout: post
title:  "openstack系列 --keystone"
date:   2016-09-16 15:01:55
tags: openstack
---


## 目录结构


[keystone 是什么 ](#A)

[keystone 的架构 ](#B)

[token 的生成方式](#C)

[可信计算部分介绍](#D)





<a name="A"></a>

## keystone 是什么

keystone 是 openstack 的认证服务模块（identy service）。 nova,glance,swift,cinder等其他服务通过keystone注册其服务的endpoint，针对这些服务的任何调用都需要经过keystone的身份认证，并获得服务的endpoint进行访问。

keystone 提供的服务可以概括为以下四个方面：
 
 - Identity:对用户身份进行验证。用户的身份凭证通常是用户名和密码。
 - Token:Identity确认完用户身份后，会给用户提供一个token以请求后续的资源。而keystone也会提供针对token的验证。token大致有两类：一类是与Tenant(也就是project)无关的token，通过这个token，可以向keystone获取Tenant列表，用户选择要访问的Tenant,然后可以获取与该Tenant绑定的token,只有通过与某个特定Tenant绑定的token才能访问此Tenant中的资源。token有自己的过期时间，如果删除某个用户的访问权限，只要删除对应token即可。
 - Catalog:Catalog服务对外提供一个服务的查询目录，即可访问的endpoint列表。
 - Policy:一个基于规则的身份验证引擎。通过配置文件来定义各种动作与用户角色的匹配关系。该部分已作为Oslo的一部分进行开发维护。

以创建虚拟机为例，keystone 的大致工作流程如下(注：以下内容摘自[<<openstack设计与实现>>](https://book.douban.com/subject/26374647/))：

![keystone process](http://7xrnwq.com1.z0.glb.clouddn.com/20160918Keystone-process.png)

 - 用户Alice发送自己的凭证到keystone，keystone认证通过后，返回给Alice一个token以及服务目录。
 - Alice通过token请求keystone查询他所拥有的Tenant列表。（如果已经知道Tenant，略过以上两步）
 - Alice选择一个Tenant，发送自己的凭证给keystone申请token，keystone验证后，返回token2。
 - Alice选择endpoint并发送token2请求创建虚拟机，keystone验证token2(包括该token是否有效，是否有权限创建虚拟机等)成功后，把请求转发给Nova，并创建虚拟机。





<a name="B"></a>

## keystone 的架构

![keystone-architecture](http://7xrnwq.com1.z0.glb.clouddn.com/20160919-openstack-keystone-architecture.png)

keystone还涉及另外一个子项目，keystonemiddleware,它提供了对token合法性进行验证的中间件。比如：客户端访问keystone提供的资源时用的是PKI类型的token，为了不必每次都需要keystone介入token的验证，我们通常会在本地节点上缓存相关证书和密钥，利用keystonemiddleware对token进行验证。

Keystone项目本身，除了后台的数据库，主要包括了一个处理restful请求的API服务进程。这些API涵盖了Identity,Token,Catalog,Policy等服务。这些不同的服务提供的功能则分别由后端Driver实现。


<a name="C"></a>

## token 的生成方式

在openstack F版本之前，token的生成方式只有UUID这一种方式（即由程序随机生成一段序列），但是在大规模的集群中，大量客户端并发请求的情况下，keystone的性能存在瓶颈（该版本还未引入keystonemiddleware）,所有的请求都要跟keystone交互。于是
PKI系统在随后的版本中引入，就可以做到本地检验而不用与keystone频繁交互。

PKI详细介绍，[check this!](https://en.wikipedia.org/wiki/Public_key_infrastructure)
此处简单介绍几个概念：

 - Certificate Auth：即CA,认证中心，数字签证的签发机构，是PKI应用中权威的，可信任的第三方机构。
 - CA私钥：CA签发的非对称加密算法中的私钥。
 - CA公钥证书：包含CA的公钥信息。
 - 签名私钥
 - 签名公钥证书

下图为UUID的token验证流程：

![uuid token valid](http://7xrnwq.com1.z0.glb.clouddn.com/20160919UUID-token-validation-flow-3.png)

下图为PKI的token验证流程：

![pki token valid](http://7xrnwq.com1.z0.glb.clouddn.com/20160919PKI-token-validation-flow-1.png)


<a name="D"></a>

## 可信计算部分介绍

在云计算环境中，可能会有成千上万个计算节点部署在不同的地方，可能有些云租户对安全的要求比较高，要求应用或虚拟机必须运行在验证为可信的节点上。openstack 在E版本时为此引入了可信计算池的概念。可信计算池的实现位于Nova项目。在FilterScheduler中加入了一个新的TrustedFilter,经过该filter的即认为是可信计算的节点。

那么可信节点是如何判定的呢？
计算节点采用基于Intel TXT 的TBoot进行可信启动，对主机的BIOS,VMM和操作系统进行完整性度量，并在得到来自认证服务的请求时将度量数据发送给认证服务。认证服务器部署基于OpenAttestation的认证服务，通过将来自主机的度量值与白名单数据库进行比对确定主机的可信状态。






## 参考文章

[openstack 设计与实现 ](https://book.douban.com/subject/26374647/)

[Understanding OpenStack Authentication: Keystone PKI](https://www.mirantis.com/blog/understanding-openstack-authentication-keystone-pki/)




***END***

