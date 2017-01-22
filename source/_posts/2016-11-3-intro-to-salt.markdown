---
layout: post
title:  "openstack系列-- 运维利器SaltStalk介绍"
date:   2016-11-3 13:06:25
tags: openstack
---

## 目录结构：

[SaltStalk 是什么](#A)

[SaltStalk 安装 ](#B)

[SaltStalk 配置 ](#C)

[SaltStalk 使用](#D)

[SaltStalk 架构](#E)


<a name="A"></a>

## SaltStalk是什么

salt官网给出的介绍简洁明了：

- 一个配置管理系统，能够维护预定义状态的远程节点(比如，确保指定的报被安装，指定的服务在运行)
- 一个分布式远程执行系统，用来在远程节点（可以是单个节点，也可以是任意规则挑选出来的节点）上执行命令和查询数据。开发其的目的是为远程执行提供最好的解决方案，并使远程执行变得更好，更快，更简单。

其实，除了saltstack,还有其他的同类型软件供我们选择，比如puppet、chef、ansible、fabric等,关于其中的区别以及优劣势可以参考这里，[Ansible vs. Chef vs. Fabric vs. Puppet vs. SaltStack](http://blog.takipi.com/deployment-management-tools-chef-vs-puppet-vs-ansible-vs-saltstack-vs-fabric/)
稍微总结下：chef,puppet 是出现时间比较早的系统，对于比较重视成熟性和稳定性的公司来说比较合适。ansible与saltstack比较适合快速灵活的生产部署环境，且适合没有太多额外要求，生产环境操作系统一致的情况下。fabric比较适合规模比较小的环境，属于入门级别的配置管理系统。



<a name="B"></a>

## SaltStalk 安装

安装比较简单，[check this](http://docs.saltstack.cn/topics/installation/index.html),不再赘述。


<a name="C"></a>

## SaltStalk 配置

saltstalk是典型的CS架构，主要分两种角色，master和minions,master负责minions的配置管理以及远程执行。一般来说，master的配置文件位于/etc/salt/master，minion的配置文件在相应机器的/etc/salt/minion,只需在minion配置文件中指定master指向即可运行。

[master的配置项](http://docs.saltstack.cn/ref/configuration/master.html)非常多，大致包括以下几项：
 - 主要配置：包括网络接口，提供服务的端口，master id,最大打开文件数，工作线程个数，返回端口，缓存目录，各种缓存配置（minions数据缓存，作业缓存等）
 - master 安全相关配置：主要是针对PKI验证的一些配置。
 - master模块管理相关配置
 - master state system 相关配置
 - pillar 相关配置
 - 日志相关配置
 - windows软件源相关配置

[minion的配置项](http://docs.saltstack.cn/ref/configuration/minion.html)跟master大致类似，不再赘述。


其他：
 - 新版saltstack 中minion有一个minion_blackout的配置，该选项设为true后，该minion不会执行除saltutil.refresh_pillar之外的其他所有命令。
 - saltstack有一个访问控制系统对于非admin用户进行细粒度的访问控制。[check this](http://docs.saltstack.cn/topics/eauth/access_control.html)
 - job管理器：可以实现对minion job的发送信号，定时调度等
 - job 返回数据管理：系统默认返回数据存储在Job cache（默认是在/var/cache/salt/master/jobs目录）中，还有其他两种存储模式：一种是External Job Cache，如下图

![External Job Cache](http://7xrnwq.com1.z0.glb.clouddn.com/20161107external-job-cache.png)

另一种是Master Job Cache，如下图：

![Master Job Cache](http://7xrnwq.com1.z0.glb.clouddn.com/20161107master-job-cache.png)

 - returner: 我们可以在master端执行远程指令时指定returner,这样minion返回的数据不仅会返回到master，也会返回到我们指定的returner。而且我们也可以按照要求实现一个returner，来替换我们的Default Job Cache。



<a name="D"></a>

## SaltStalk 使用




<a name="E"></a>

## SaltStalk 架构





## 参考文章

[v2ex](https://www.v2ex.com/t/316723#reply21)

[stackoverflow-Preventing multiple process instances on Linux](http://stackoverflow.com/questions/2964391/preventing-multiple-process-instances-on-linux)

[Linux 编程中的文件锁之 flock](http://blog.jobbole.com/102538/)

[Linux 2.6 中的文件锁](https://www.ibm.com/developerworks/cn/linux/l-cn-filelock/)

[Python wiki](https://wiki.python.org/moin/DbusExamples)




***END***
