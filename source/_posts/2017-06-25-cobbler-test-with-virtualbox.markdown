---
layout: post
title:  "Devops-- 借助virtualbox 初次实践 cobbler"
date:   2017-06-26 00:16:11
tags: 
  - ops
  - devops
  - cobbler
---

## Cobbler 简介

Cobbler 是由Python 语言开发，实现在网络安装环境中快速的安装linux 系统。
在没有出现Cobbler之前，我们通常是通过使用Kickstart 来进行自动化安装操作系统，但是kickstart 并未实现完全的自动化，我们依然需要安装配置各种服务（DHCP,TFTP,HTTP等），编写kickstart 脚本（即ks.cfg文件）等等。
Cobbler可以看做是在kickstart项目上的一层封装，融合了DHCP,TFTP，PXE等一系列服务，提供了CLI和web两种管理形式，使得网络安装操作系统更加方便，同时也提供了API接口方便二次开发。

##  Cobbler 原理概述

Cobbler 是典型的CS架构，我们只需要在服务端做好相应的安装配置，客户端设置PXE网络启动即可。整体的运作流程如下，图片来自[Cobbler自动化安装配置实践](https://wsgzao.github.io/post/cobbler/):

![cobbler-process](http://oeptotikb.bkt.clouddn.com/2017-06-25-cobbler-process.jpeg)

## Cobbler 实践

本次试验借助 VirtualBox 实现：

### 准备工作

- 下载 centos7-minimal 镜像，并使用该镜像开启一个虚拟机guest#1
- 注意网络配置 两个网卡，一个是bridge模式，便于利用xshell等工具连接，一个是设为内部网络。
- 安装完毕后，设置第二个网卡的静态IP，不要与第一个网卡的网络重复，之后会用第二个网卡所在的局域网进行网络安装。

### Cobbler 安装

参照 官网[Red Hat Entperise Linux](https://cobbler.github.io/manuals/2.6.0/2/2/2_-_RHEL_and_CentOS.html)

注：安装需要用到 epel源，因为是Centos，只需 yum install epel-release 即可。

### Cobbler 配置

参考官网[Cobbler Quickstart Guide](https://cobbler.github.io/manuals/quickstart/)

注：

- 利用cobbler check 查看配置错误，有些地方并非必须修改。
- server 与 next-server设置为第二块网卡的ip

### Cobbler 使用

- 利用lrzsz命令上传 centos7-minimal 镜像至虚拟机guest#1.
- 本地挂载镜像
```bash
mount -o loop ./CentOS-7-x86_64-Minimal-1611.iso /mnt
```
- 导入镜像
```bash
cobbler import --name=centos7 --arch=x86_64 --path=/mnt
```

- 执行 cobbler sync命令
- virtualbox 开启另一个虚拟机，一个网卡，设置为内部网络，adaptor type 为PC-Net III （为了PXE boot）。设置启动方式为网络启动优先。


### 遇到的问题

- TFTP Timed out : guest#1的TFTP Server 没有启动造成。
- failed to start switch root：[dracut cant locate /dev/root](https://www.centos.org/forums/viewtopic.php?t=57419)

## 参考文章


[Cobbler doc](https://cobbler.github.io/)

[cobbler wiki](https://zh.wikipedia.org/wiki/Cobbler_(%E8%BD%AF%E4%BB%B6))

[Cobbler自动化安装配置实践](https://wsgzao.github.io/post/cobbler/)

[自动化运维工具Cobbler](http://cuchadanfan.blog.51cto.com/9940284/1698348)

[Testing out cobbler with virtuabox](http://www.webscalability.com/blog/2013/03/testing-out-cobbler-with-virtuabox/)

 ***END***