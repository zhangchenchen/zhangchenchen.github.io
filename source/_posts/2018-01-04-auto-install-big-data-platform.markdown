---
layout: post
title:  "Ops-- CDH大数据平台自动化安装部署"
date:   2018-01-04 18:00:10
tags: 
  - Ops
  - Cobbler
  - SaltStalk
---


## 概述

Cloudera 的CDH应该是目前大数据平台安装部署的首选，不过CDH只是简化了大数据组件的安装部署，其他的比如操作系统的安装，磁盘raid，seed节点配置，免密登录，ntp安装等都是需要手动或脚本完成，所以这里为了安装部署的方便，便把这一系列过程自动化，实现CDH大数据平台的自动化安装。

使用的工具有：
- KickStart
- Cobbler
- SaltStalk
- CdhBoot(自研的基于saltstalk的web 平台)


## 安装流程解析

大致有以下几个过程：
- 磁盘raid（可选）
- seed节点操作系统安装
- node节点操作系统安装
- cdh准备安装工作
- cdh安装

现在就按这几个顺序大致记录下整过程所需做的工作。

### 磁盘raid

规划如下：

- datenode 系统分区用两块磁盘做raid1。
- datenode的数据存储分区用单独一块磁盘做raid0（可选，单独一块磁盘不做raid也是可以的）
- namenode 系统分区用两块磁盘做raid1。
- namenode 数据存储分区做raid5。

磁盘raid这一块因为服务器硬件的不同，做raid方式也不同，博主一直没能找到一个比较通用的自动化方式，所以最终只能手动解决。


### seed节点操作系统安装

这一部分的工作主要是seed节点操作系统的制作，大致步骤如下：
- 安装镜像制作软件包（createrepo & mkisofs
）
- 拷贝系统镜像文件到指定目录
- 编写Kickstart 文件
- 生成依赖关系和ISO文件

具体操作可以参考[基于Kickstart自动化安装CentOS实践](https://wsgzao.github.io/post/kickstart/#)

这里注意：
1. 因为是离线环境，所以这里要把所需要安装的所有rpm包全部放到镜像Packages目录里。
2. ks文件的编写，可以先手动安装一个系统，然后在安装完成的root目录下会生成一个anaconda-ks.cfg，该文件记录了你手动安装时的操作，然后根据这个文件去更改。这里有一个示例[Centos-KickStart-Example](https://github.com/zhangchenchen/Centos-KickStart-Example)。
3. seed节点安装系统后，需要手动配置一下seed节点的IP ，这里也可以写入ks文件，但相对不灵活，所以还是建议单独配置。


### node节点操作系统安装

该部分就是要用cobbler去推操作系统，需要我们在seed节点安装cobbler，导入镜像，然后配置待安装node节点的mac地址，最后推系统。为了完成自动化，我们是这样做的：

- 在seed节点安装的ks文件中，我们在post部分加入一个脚本或者salt命令，在seed节点安装并启动CdhBoot。
- Cdhboot的web页面配置node mac地址，同时，在后端接收到请求后会调用salt命令进行cobbler的安装部署以及image的导入。

这里解释下为什么不把cobbler的安装配置放在ks文件中，因为cobbler依赖我们配置的网卡IP，如果那个网卡没起来，cobbler就会报错，所以我们把cobbler的部署放在配置ip 后。
注意，利用cobbler推送的镜像中也要编写相应的ks文件。

### cdh准备安装工作

该部分主要是Cdhboot 调用salt api 完成免密登录，ntp安装，yum源设定等等准备工作

### cloudera-manager安装

同上，这部分也是Cdhboot 调用salt api 完成。


这样，我们的工作大致完成，完成之后，大数据平台的安装步骤如下：

- 各节点做raid
- 安装seed节点,配置IP
- 在Cdhboot 界面完成node操作系统安装，cdh准备工作，cloudera-manager安装。
- 在cloudera-manager界面完成cdh安装。

## 参考文章

[基于Kickstart自动化安装CentOS实践](https://wsgzao.github.io/post/kickstart/#)

[Linux Kickstart 自动安装](http://liaoph.com/linux-kickstart/)

[SaltStack事件驱动(4) – event reactor ](https://www.centos.bz/2016/12/saltstack-event-reactor/)

[namenode datanode 是否有必要做 raid](https://groups.google.com/forum/?hl=lt_US&fromgroups#!topic/hadoopors/ekHIDDupnI0)

[解决PXE批量安装Linux系统时kickstart自动识别硬盘名称的问题](http://blog.51cto.com/1130739/1757208)

***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***