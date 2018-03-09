---
layout: post
title:  "Docker-- Docker storage driver 概述"
date:   2018-03-09 11:57:10
tags: 
  - Docker
---


## 概述

Docker 配置的时候有一个很重要的配置项就是 storage driver选项，本篇博客详细介绍下storage driver这一配置项的相关内容。

## 背景

首先是 storage driver出现的原因。我们知道容器的存储大致有两种，一种是在容器外的，比如 volume，不会随着容器的消亡而消失，有自己的生命周期。还有一种是容器内的，这种存储跟对应容器的生命周期是紧密结合在一起的。而我们要说的就是容器内的存储。
本地的Docker引擎有一个Docker镜像层的缓存，镜像层是层层叠加的。当容器运行起来的时候就是在镜像层上起来的。基于同一个镜像运行的容器会共用一个镜像。那如何保证容器内操作后内容的独立呢，就要使用容器层以及写时复制（COW）技术。如下图：

![cow](http://oeptotikb.bkt.clouddn.com/20180309142857-cow.jpg)

最底层的基础镜像是ubuntu的系统镜像，再往上是分层的镜像层（比如dockerfile中的软件安装等），这些都是只读的镜像层。最上面才是可以读写的容器层，不同的容器有不同的容器层，共用相同的镜像层。当某个容器需要写操作时，会先将写的内容从镜像层复制到容器层，然后再写入（也就是写时复制），读的时候会从容器层开始，如果命中则读取，没有命中则依次往下读取。

![storage-driver](http://oeptotikb.bkt.clouddn.com/20180309145042-storage-driver.jpg)

至于以上原理的具体的实现，就是storage driver做的事。

## storage driver 的种类以及选型

注：本文是基于最新版Docker（V17.12），版本不同，选择也不同，具体参考官网。

目前为止，常用的storage driver有以下几种：AUFS，Btrfs，Device mapper，Overlayfs，ZFS，VFS。其中，
 - VFS是接口的“原生”的实现，完全没有使用联合文件系统或者写时复制技术，而是将所有的分层依次拷贝到静态的子文件夹中，然后将最终结果挂载到容器的根文件系统。它并不适合实际或者生产环境使用，但是对于需要进行简单验证的场景，或者需要测试Docker引擎的其他部件的场景，是很有价值的。
 - Btrfs和ZFS针对特定的文件系统，也就是 backing filesystem必须相应的是Btrfs和ZFS。
 - AUFS和Overlayfs（包括Overlayfs2）都是在原有文件系统上基于联合挂载实现，而Device mapper是所有的镜像和容器存储在它自己的虚拟设备上，这些虚拟设备是一些支持写时复制策略的快照设备。

具体的选择策略要根据linux 发行版，docker版本以及文件系统来决定：

对于Docker 社区版本来说，不同linux发行版的选择如下：

![linux-distribution](http://oeptotikb.bkt.clouddn.com/20180309151325-linux-distribution.jpg)

对于不同的文件系统，推荐如下：

![file-system](http://oeptotikb.bkt.clouddn.com/20180309151640-file-system.jpg)

综上，选择时可以如下选择：

- 1，如果内核支持多个storage driver，可以如下考虑优先级：首先考虑特定文件系统的storage driver,即Btrfs和ZFS。否则，可以使用稳定和通用的配置，overlay2,或device mapper（需要手动配置direct-lvm，因为默认的loopback-lvm性能一般）。
- 2，依据具体的Docker 版本, 操作系统版本做选择（见上图）。
- 3，依据具体的 backing filesystem做选择（见上图）。



## 几种典型 storage driver的原理

简单讲下AUFS，Overlayfs，Device mapper实现的具体原理。


### AUFS 的实现原理

AUFS是一种联合文件系统，意思是它将同一个主机下的不同目录堆叠起来(类似于栈)成为一个整体，对外提供统一的视图。AUFS是用联合挂载来做到这一点。 在Docker中，AUFS实现了镜像的分层。AUFS中的分支对应镜像中的层。

![aufs](http://oeptotikb.bkt.clouddn.com/20180309155232-aufs.jpg)

- aufs中文件的读写:读的时候会先去container layer读，如果没有会继续往下层读取。写的时候也是，如果container layer文件没有，先去image layer复制文件到container layer,然后再写入。因为AUFS工作在文件的层次上，也就是说AUFS对文件的操作需要将整个文件复制到读写层内，哪怕只是文件的一小部分被改变，也需要复制整个文件。
- aufs中文件的删除:AUFS通过在最顶层(container layer)生成一个whiteout文件来删除文件。whiteout文件会掩盖下面只读层相应文件的存在，但它事实上没有被删除。

### Overlayfs的实现原理

OverlayFS与AUFS相似，也是一种联合文件系统(union filesystem)，与AUFS相比，OverlayFS： 设计更简单，被加入Linux3.18版本内核 ，可能更快。

Overlay通过三个概念来实现它的文件系统：一个“下层目录（lower-dir）”，一个“上层目录（upper-dir）”，和一个做为文件系统合并视图的“合并（merged）”目录。受限于只有一个“下层目录”，需要额外的工作来让“下层目录”递归嵌套（下层目录自己又是另外一个overlay的联合），或者按照Docker的实现，将所有位于下层的内容都硬链接到“下层目录”中，这就可能导致inode爆炸式增长（因为有大量的分层内容和硬连接）。

![overlay](http://oeptotikb.bkt.clouddn.com/20180309160513-overlay.jpg)

Overlay2基于Linux内核4.0和以后版本中overlay的特性，可以允许有多个下层的目录，解决了一些因为最初驱动的设计而引发的inode耗尽和一些其他问题。不过由于代码库相对还比较年轻，有待时间的检验。

理论情况下，overlay2 和 overlay要比aufs 和 devicemapper性能好，甚至，一些情况下，overlay2要比btrfs好。不过也有一些需要注意的方面：
- OverlayFS 支持页缓存（page caching）共享。意味着多个使用同一文件的容器可以共享同一页缓存，这使得overlayfs具有很高的内存使用效率。
- 同aufs一样，第一次写文件时需要复制整个文件，这会带来一些性能开销，在修改大文件时尤其明显。 
- overlay的inode限制。

### Device mapper的实现原理

device mapper将所有的镜像和容器存储在它自己的虚拟设备上，这些虚拟设备是一些支持写时复制策略的快照设备。device mapper工作在块层次上而不是文件层次上，这意味着它的写时复制策略不需要拷贝整个文件。 
device mapper创建镜像的过程如下： 
- 使用device mapper的storge driver创建一个精简配置池；精简配置池由块设备或稀疏文件创建。 
- 接下来创建一个基础设备； 
- 每个镜像和镜像层都是基础设备的快照；这写快照支持写时复制策略，这意味着它们起始都是空的，当有数据写入时才耗费空间。
![device-mapper](http://oeptotikb.bkt.clouddn.com/20180309162425-device-mapper.jpg)
镜像的每一层都是它下面一层的快照，镜像最下面一层是存在于thin pool中的base device的快照。容器是创建容器的镜像的快照。
device mapper跟之前的storage driver最大的不同就是它是基于块而不是基于文件，所以对文件的操作实际是对对应文件块的操作，默认每个块的大小为64KB。
图展示了容器中的某个进程读取块号为0x44f的数据： 
![device-mapper](http://oeptotikb.bkt.clouddn.com/20180309163531-device-mapper.jpg)

device mapper不是最有效使用存储空间的storage driver，启动n个相同的容器就复制了n份文件在内存中，这对内存的影响很大。所以device mapper并不适合容器密度高的场景。

## 参考文章

[Docker storage drivers](https://docs.docker.com/storage/storagedriver/select-storage-driver/)

[Docker用户指南(4) – 存储驱动选择](https://www.centos.bz/2016/12/select-a-docker-storage-driver/)

[深入了解Docker存储驱动](http://dockone.io/article/1765)

[Docker之几种storage-driver比较](http://blog.csdn.net/vchy_zhao/article/details/70238690)


***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***