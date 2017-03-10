---
layout: post
title:  "linux-- 利用LVM 对虚拟机进行手动扩容"
date:   2017-03-10 11:13:46
tags: 
  - linux
  - lvm
  - openstack
---



## 引出

一般来说，如果openstck镜像中安装了growpart(之前的版本叫做growroot),在创建虚拟机的时候就会实现自动扩容。但是今天碰到几个虚拟机没有实现自动扩容的情况，需要手动进行扩容，用到lvm的一些知识，这里记录一下。


## 关于LVM的一些知识

LVM利用Linux内核的device-mapper来实现存储系统的虚拟化（系统分区独立于底层硬件）。通过LVM，你可以实现存储空间的抽象化并在上面建立虚拟分区，可以更简便地扩大和缩小分区，可以增删分区时无需担心某个硬盘上没有足够的连续空间。

简单来说：LVM是Linux环境中对磁盘分区进行管理的一种机制，是建立在硬盘和分区之上、文件系统之下的一个逻辑层，可提高磁盘分区管理的灵活性。

LVM的基本组成块如下：

 - 物理卷Physical volume (PV)：可以在上面建立卷组的媒介，可以是硬盘分区，也可以是硬盘本身或者回环文件（loopback file）。物理卷包括一个特殊的header，其余部分被切割为一块块物理区域（physical extents）。 
 - 卷组Volume group (VG)：将一组物理卷收集为一个管理单元。
 - 逻辑卷Logical volume (LV)：虚拟分区，由物理区域（physical extents）组成。

![lvm-pic](http://7xrnwq.com1.z0.glb.clouddn.com/2017-03-10-lvm-500x290.jpg)


## 对虚拟机进行手动扩容

注：本文测试为实验环境，直接利用workstation 开启虚拟机，扩展磁盘，最后实现手动扩展。

首先，确认磁盘使用lvm以及确定磁盘大小：

![lvm-1](http://oeptotikb.bkt.clouddn.com/2017-030-10-lvm-1.png)

可以看到磁盘/dev/sda大小总共约为53G，/dev/sda2是已经加入卷组的一个物理卷。

![lvm-2](http://oeptotikb.bkt.clouddn.com/2017-03-10-lvm-2.png)

/dev/mapper/cl-root 与/dev/mapper/cl-swap 设备其实是逻辑卷。
![lvm-3](http://oeptotikb.bkt.clouddn.com/2017-03-10-lvm-3.png)


接下来在workstation面板进行磁盘扩展：

![lvm-4](http://oeptotikb.bkt.clouddn.com/2017-03-10-lvm-4.png)

查看磁盘情况：

![lvm-5](http://oeptotikb.bkt.clouddn.com/2017-03-10-lvm-df.png)

可以看到磁盘已经扩展为60G+了，但是磁盘分区还是两个且大小没变，而且挂载root的/dev/mapper/cl-root依然是50G,所以接下来我们需要做的就是将多出来的10G空间进行磁盘分区并最终扩展到逻辑卷dev/mapper/cl-root。

首先磁盘分区并将该分区设置为LVM格式。

![lvm-6](http://oeptotikb.bkt.clouddn.com/2017-03-10-lvm-6.png)

重启机器，将/dev/sda3创建物理卷，并将该物理卷加入卷组。

![lvm-7](http://oeptotikb.bkt.clouddn.com/2017-03-10-lvm-7.png)

扩展逻辑卷并将对应的文件系统扩展，如果是ext3/4文件系统用resize2fs 命令。

![lvm-8](http://oeptotikb.bkt.clouddn.com/2017-03-10-lvm-8.png)

扩展成功。

![lvm-9](http://oeptotikb.bkt.clouddn.com/2017-03-10-lvm-9.png)



## 参考文章


[LVM WIKI](https://wiki.archlinux.org/index.php/LVM_)

[LVM逻辑卷管理配置小结](https://wsgzao.github.io/post/lvm/)

[Linux LVM逻辑卷配置过程详解](http://dreamfire.blog.51cto.com/418026/1084729)

[How to Increase the size of a Linux LVM by expanding the virtual machine disk](https://www.rootusers.com/how-to-increase-the-size-of-a-linux-lvm-by-expanding-the-virtual-machine-disk/)


***END***
