---
layout: post
title:  "ceph-- 使用国内yum源搭建ceph集群"
date:   2017-02-28 16:13:46
tags: ceph
---



## 引出

初衷是自己搭建一套ceph的实验集群，按照官方的quick start,按理说应该是分分钟的事，因为GFW的原因，搞得忙活了一上午，再加上心情烦躁,中间出现了各种差错，这里记录一下。


## 系统预检并安装ceph-deploy

准备集群系统，网络设置，SSH无密码登录等设置略过，详细参考[PREFLIGHT CHECKLIST](http://docs.ceph.com/docs/master/start/quick-start-preflight/)

这里有一个地方需要注意的是：centos7 在修改HOSTNAME 的时候命令如下，无需重启：

```bash
$ sudo hostnamectl --static set-hostname <host-name>
```

预检完成后，ceph-deploy 的安装以及 ceph在各节点的安装如果出现错误的话很大可能上是yum 源的问题，所以重点说下如何配置yum国内源。


### 更换国内 YUM 源

- 更换base源为国内163源:

```bash
$ wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.163.com/.help/CentOS7-Base-163.repo
```

- 更换epel源为中科大的源(注：随着版本更迭，以下链接可能会失效，如果出现404错误，那就需要更改链接)：

```bash
$ rpm -Uvh  http://mirrors.ustc.edu.cn/centos/7/extras/x86_64/Packages/epel-release-7-9.noarch.rpm

$ rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7

```

- 更换ceph源为国内163源

建议不用改变ceph.repo,直接设置环境变量即可，如果更改ceph.repo，那么执行ceph-deploy的时候还是需要在命令行中指定国内源url，否则又会使用官方源。

```bash
$ export CEPH_DEPLOY_REPO_URL=http://mirrors.163.com/ceph/rpm-jewel/el7

$ export CEPH_DEPLOY_GPG_URL=http://mirrors.163.com/ceph/keys/release.asc

$ yum clean all

$ yum makecache

$ yum update -y

```






## 安装并配置ceph cluster

接下来的事情就跟官网的步骤差不多了，大致如下：

```bash
# Create monitor node
ceph-deploy new node1 node2 node3

# Software Installation
ceph-deploy install deploy node1 node2 node3

# Gather keys
ceph-deploy mon create-initial

# Ceph deploy parepare and activate
ceph-deploy osd prepare node1:/dev/sdb node2:/dev/sdb node3:/dev/sdb
ceph-deploy osd activate node1:/var/lib/ceph/osd/ceph-0 node2:/var/lib/ceph/osd/ceph-1 node3:/var/lib/ceph/osd/ceph-2

# Make 3 copies by default
echo "osd pool default size = 3" | tee -a $HOME/ceph.conf

# Copy admin keys and configuration files
ceph-deploy --overwrite-conf admin deploy node1 node2 node3

```


## 一个尚未解决的顽疾

在公司使用国内源安装ceph的时候，出现了一个这样的错误：

```bash
 http://mirrors.163.com/ceph/rpm-jewel/el7/x86_64/ceph-mds-10.2.5-0.el7.x86_64.rpm: [Errno -1] Package does not match intended download. Suggestion: run yum --enablerepo=ceph clean metadata
```

Package does not match intended 这个错误在我安装其他软件的时候也出现过，用尽了各种办法也没有解决该问题，但我在家里的时候却没有出现过该问题。所以推测是公司网络用了类似缓存的机制。

## 更新-2017-3-18

找到了解决上述问题的方法，就是先直接wget下来rpm包，然后用yum localinstall 或者rpm -ivh 安装,如果出现两个包互相依赖的情况，就两个包同时安装。

```bash
wget http://mirrors.163.com/ceph/rpm-jewel/el7/x86_64/ceph-mds-10.2.5-0.el7.x86_64.rpm

yum localinstall ceph-mds-10.2.5-0.el7.x86_64.rpm
```


## 参考文章


[PREFLIGHT CHECKLIST](http://docs.ceph.com/docs/master/start/quick-start-preflight/)

[STORAGE CLUSTER QUICK START](http://docs.ceph.com/docs/master/start/quick-ceph-deploy/)

[如何使用国内源部署Ceph？](http://ceph.org.cn/2016/09/02/%E5%A6%82%E4%BD%95%E4%BD%BF%E7%94%A8%E5%9B%BD%E5%86%85%E6%BA%90%E9%83%A8%E7%BD%B2ceph%EF%BC%9F/)

[CentOS7修改主机名](http://www.centoscn.com/CentOS/config/2014/1031/4039.html)

[CentOS 7 x86_64适用的EPEL安装源 国内镜像列表](http://itgeeker.net/centos-7-epel-china-mirror-repository/)


***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***
