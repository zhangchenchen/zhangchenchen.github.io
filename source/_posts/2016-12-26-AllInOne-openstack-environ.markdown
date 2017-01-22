---
layout: post
title:  "openstack系列--AllInOne openstack 实验环境搭建"
date:   2016-12-26 20:57:25
tags: openstack
---

## 目录结构：

[虚拟机搭建](#A)

[利用RDO 搭建allinone openstack 环境](#B)

[利用Fuel 搭建openstack 环境](#C)



<a name="A"></a>

## 虚拟机搭建



1. virtualbox 下载安装，centos 7下载
2. 利用virtualbox启动centos 7.遇到问题如下：
    - 网络一直连不通，后来参考这篇文章[VirtualBox下虚拟机和主机内网互通+虚拟机静态IP的网络配置](http://xintq.net/2014/09/05/virtualbox/)
    - 不过我只用了一个虚拟网卡就够用了，因为只建一个虚拟机，不用虚拟机之间联通。可以使用nat，或者桥接模式。最好使用桥接模式，因为可以使用客户端SSH连接。如果网络没通的话可能还需要配置一下网络参数，比如IP地址，dns地址等。
    


<a name="B"></a>

## 利用RDO 搭建allinone openstack 环境

1. 搭建openstack实验环境大致有三种：RDO的packstack,Mirantis 的Fuel安装，Devstack。这三种方式所有的安装过程都不难，但是因为网络的问题，总会出现很多意料之外的事情。总体来说，RDO这种allinone的方式是最简单的，就算出了问题，80%的可能出在网络上。
2. 按照[quick start](https://www.rdoproject.org/install/quickstart/)进行安装。
3. 出现的问题：
    - 安装 yum install -y openstack-packstack 时出现“Package does not match intended download” ，这个问题google看了好几页一直没解决，后来我在家里的时候用家里的网络就可以了，估测是公司的网络用了代理，存在缓存的原因。
    - 最后执行 packstack --allinone 时经常卡住，一般卡在 testing if puppet apply is finished 的居多，其实也是网络问题，我在深夜网速较好的时候成功率比较高。

4. 搭建完成后就可以在虚拟机上进行调试了，推荐一个vim 的设置[A set of vim, zsh, git, and tmux configuration files.(*nix开发环境一键配置）](https://github.com/int32bit/dotfiles)


<a name="C"></a>

## 利用Fuel 搭建openstack 环境

 - 目前最新版本还是openstack M版。主要是参考这篇文章[部署安装Mirantis OpenStack Fuel 9.0](http://blog.csdn.net/titan0427/article/details/51982609)。
 - 没有部署成功，fuel server 部署成功了，但是最终部署openstack的时候死活过不去，上篇文章的作者也说可重复性不高，看报错日志是download error，也是网络问题。以后有机会大半夜再搞一发。
 - 上篇文章没有提到的一件事是 在fuel server部署完成后，ubuntu镜像最好是自己下载然后上传到 Fuel server,设置resposity 的时候设置为本地源，这样可以通过网络验证。 





## 参考文章

[部署安装Mirantis OpenStack Fuel 9.0](http://blog.csdn.net/titan0427/article/details/51982609)

[quick start](https://www.rdoproject.org/install/quickstart/)


[A set of vim, zsh, git, and tmux configuration files.(*nix开发环境一键配置）](https://github.com/int32bit/dotfiles)



***END***
