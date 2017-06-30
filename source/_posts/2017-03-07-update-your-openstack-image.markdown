---
layout: post
title:  "openstack-- 如何升级现有镜像"
date:   2017-03-07 13:13:46
tags: 
  - openstack
  - ceph
---



## 引出

公司云平台有些镜像的密码注入出了点问题，问题的原因初步诊断为镜像问题，需要对这些镜像进行cloudinit/cloudbase的升级操作。对于已知密码的镜像，可以直接开启一个虚拟机，完成升级后，对该虚拟机做snapshot 操作（nova image-create）即可。对于不知道密码的镜像，经过一番搜索后，可以通过将镜像挂载到本地宿主机，然后chroot的方法进行升级,后来又从官网了解到利用libguestfish 的方法。以下为一些简单记录。


## 利用 nova image-create 升级镜像 

- 利用镜像开启一个虚拟机
- 进入虚拟机做升级操作，并删除/var/lib/cloud 下的文件，windows需删除LocalScripts/下的文件。
- 完成操作后，退出虚拟机并对该虚拟机做snapshot操作。
- 镜像完成。


## 利用 本地挂载的方法升级镜像

如果忘记密码的话我们可以使用该方法进行镜像的升级，不过因为glance后端存储方式的不同，挂载到本地HOST的方法也不同，下文分别以本地存储，ceph存储讲解。


### glance 存储为本地存储


#### 镜像格式为raw格式且镜像文件系统中使没有用LVM

```bash
$ fdisk -lu cirros-0.3.0-x86_64-disk.raw  # 查看该镜像的分区情况以及是否使用LVM

$ kpartx -av cirros-0.3.0-x86_64-disk.raw #读取镜像的分区表，然后生成代表相应分区的设备/dev/loop0 ，/dev/loop1等等

$ mount /dev/mapper/loop0p1 /mnt/image # 将需要挂载的分区设备进行挂载，此时进入/mnt/image 应该就可以看到熟悉的系统文件目录了 

$ cd /mnt/image && cp /etc/resolv.conf etc/resolv.conf # 覆盖dns

$ chroot /mnt/image bash  #chroot到镜像文件中

$ source /etc/profile && source ~/.bashrc #初始化环境变量

```

然后就可以进行升级操作了，操作完成后，需要将临时文件清除，yum clean all 等。最后退出chroot 并进行unmount 操作：

```bash
$ exit # 退出chroot

$ umount /mnt/image 

$ kpartx -d cirros-0.3.0-x86_64-disk.raw  # 删除设备映射关系

```


#### 镜像格式为raw格式且镜像文件系统中使用LVM

```bash
$ fdisk -lu centos7-x86_64.img  # 查看该镜像的分区情况以及是否使用LVM

$ kpartx -av centos7-x86_64.img # 读取镜像的分区表，然后生成代表相应分区的设备/dev/loop0 ，/dev/loop1等等

$ pvscan # 查找lvm设备并记下vg的名字示例为centos

$ vgchange -ay centos # 激活vg中的lvm

$ lvs # 查看lvm设备列表

$ mount /dev/centos/root /media/ #挂载对应的lvm

$ cd /media/ && cp /etc/resolv.conf etc/resolv.conf

$ chroot /mnt/image bash  #chroot到镜像文件中

$ source /etc/profile && source ~/.bashrc #初始化环境变量

```

退出chroot 并进行unmount 操作如下：

```bash
$ exit  # 退出chroot

$ umount /media/ 

$ vgchange -an centos # deactive vg 中的lvm

$ kpartx -d  centos7-x86_64.img # 删除设备映射关系 

```

如下为一个完整实例（省略chroot操作）：

![chroot-lvm](http://7xrnwq.com1.z0.glb.clouddn.com/2017-03-08-chroot-lvm.png)


#### 镜像格式为qcow2格式

qcow2 格式的镜像需要使用qemu自带的一个工具qemu-nbd，该工具会生成一个nbd(network block device)，然后将镜像映射到该nbd,便可以像普通block设备一样挂载了。

```bash
$ modinfo nbd # 查看系统kernel是否支持nbd模块

$ modprobe nbd max_part=16 # 加载 nbd模块

$ lsmod | grep nbd # 查看nbd模块是否加载

$ qemu-nbd -c /dev/nbd0 centos7.qcow2 # 将qcow2镜像映射为网络块设备(nbd)

$ ll /dev/nbd0* # 查看映射情况

$ mount /dev/nbd0p1 /media/ # 挂在对应的nbd设备

```

卸载的步骤如下：

```bash
$ umount /media

$ qemu-nbd -d /dev/nbd0

```

如果该qcow2镜像使用LVM的话，前半段挂载nbd设备，后半段参照raw格式。



### glance 存储为ceph存储

首先需要知道ceph作为glance存储后端的时候，openstack会充分利用ceph 的分层clone的特性来支持快速创建虚拟机。镜像会首先生成一个以snap结尾命名的快照，以后每次开启一个虚拟机都会clone该snapshot。该snapshot是protected，所以无法更改，而如果直接修改image的话，无法同步对应的snapshot。所以最稳妥的方法是先复制一个rbd image，然后修改这个复制的image，修改成功后覆盖或直接删除之前的image。

```bash
$ rbd  cp $POOL/${IMAGE_ID} $POOL/${IMAGE_ID}_copy # 复制image （允许跨pool）

$ rbd map $POOL/${IMAGE_ID}_copy # 挂载rbd镜像到本地（注意kernel的支持）

$ mount /dev/rbd0p1 /mnt # 挂载对映的rbd到挂载点(rbd0p1 只是示例，依据真实情况填写)

$ ll /mnt 

```

之后就可以执行chroot 以及镜像的更新操作了。
之后的umount操作如下：
```bash
$ umount -r -l /mnt/ 

$ rbd unmap /dev/rbd0 #从rbd中卸载

```


再然后就是更新glance了，大致步骤如下：

```bash
$ glance image-create --name NewImage # 创建image，占个坑，不上传，得到new imageID
 
$ rbd mv  $POOL/${IMAGE_ID}_copy $POOL/${NEW_IMAGE_ID} # rbd改名

$ rbd --pool=$POOL --image=${NEW_IMAGE_ID} --snap=snap snap create # 创建snapshot

$ rbd --pool=$POOL --image=${NEW_IMAGE_ID} --snap=snap snap protect #对snapshot做保护

$ glance image-update --name="$DISPLAY_NAME" --disk-format=raw --container-format=bare --is-public=True ${NEW_IMAGE_ID} # 更新glance 元数据

$ glance image-update --property image_meta="$image_meta" ${NEW_IMAGE_ID}

$ glance image-update --property hw_qemu_guest_agent=yes ${NEW_IMAGE_ID}

$ glance image-update --location rbd://$FSID/$POOL/$NEW_IMAGE_ID/snap ${NEW_IMAGE_ID} #更新镜像的地址

```


## 使用 libguestfs 修改虚拟机镜像


### libguestfs项目介绍

libguestfs 其实是一系列工具组成的，目的就是为了连接并修改本地虚拟机镜像。可以实现的功能有很多，包括：修改镜像内文件，脚本，查看镜像文件系统容量使用情况，物理机与虚拟镜像之间的文件传递，备份，克隆，甚至新建虚拟机实例，格式化磁盘,修改磁盘大小等等。参考[libguestfs ](http://libguestfs.org/)

### 部分示例

安装比较简单，如果是rh/centos系列，直接yum安装：

```bash
$ yum install -y libguestfs-tools
```

下面主要分三部分guestfish，guestmount 以及virt-* tools 讲解

#### guestfish 讲解与示例

通俗讲，guestfish 的作用就是修改镜像内的文件。它并不会将镜像文件系统直接挂载到本地，而是提供了一个类似shell的交互接口允许你查看，编辑，删除文件。

示例：

```bash
$ guestfish --rw -a centos63.raw  # 进入guestfish shell ,Mount the image in read-write mode as root

```
进入之后 ，先执行run ，会创建一个虚拟机实例。

![libguestfs](http://oeptotikb.bkt.clouddn.com/2017-03-24-libguestfs.png)

如果出现错误的话，可以 export LIBGUESTFS_DEBUG=1 打开debug模式查找错误。或者利用 libguestfs-test-tool 命令测试一下。
接下来示例编辑网卡配置：

```bash
><fs> list-filesystems  #列出可挂载的文件系统

><fs> mount /dev/vg_centosbase/lv_root /  #挂载

><fs> edit /etc/sysconfig/network-scripts/ifcfg-eth0 #编辑文件

><fs> exit #退出
```


#### guestmount 讲解与示例

guestmount 可以实现直接将镜像的文件系统挂载到本地。示例如下：

```bash
$ guestmount -a centos63_desktop.qcow2 -i --rw /mnt # i 参数表示自动查找root分区并挂载

$ rpm -qa --dbpath /mnt/var/lib/rpm  #示例查看rpm包

$ umount /mnt 
```

若umount失败（通常报错 device is busy），可以使用lazy umount，加 l 参数，即：

```bash
$ umount /mnt -l
```



#### virt-* tools 讲解与示例

大概以下几种工具：

- virt-edit : 修改镜像内的文件。
- virt-df : 查看镜像磁盘占用情况。
- virt-resize : resize 镜像。
- virt-sysprep : 准备发布镜像前的一系列操作（比如删除SSH HOST，删除mac地址，删除user 信息）
- virt-sparsify : 镜像稀疏（消除镜像空洞）
- virt-p2v : 物理机转换为虚拟机（kvm）.
- virt-v2v : xen或vmware镜像转换为kvm镜像。

几个示例（来自[openstack doc](https://docs.openstack.org/image-guide/modify-images.html)）:

第一个示例是修改镜像文件：

```bash
$ virsh shutdown instance-000000e1

$ virt-edit -d instance-000000e1 /etc/shadow  # d means domain

$ virsh start instance-000000e1

```

第二个示例是 resize image：

```bash
$ virt-filesystems --long --parts --blkdevs -h -a /data/images/win2012.qcow2 #查看分区

$ qemu-img create -f qcow2 /data/images/win2012-50gb.qcow2 50G #新建一块qcow2 image 空间

$ virt-resize --expand /dev/sda2 /data/images/win2012.qcow2 /data/images/win2012-50gb.qcow2 # 扩展空间

```





## 更新

 - 2017-03-23 增加 libguestfs 修改虚拟机镜像的方法。


 - 2017-03-34 增加 libguestfs 部分示例




## 参考文章


[Modify images](https://docs.openstack.org/image-guide/modify-images.html)

[在线升级glance镜像技巧](http://int32bit.me/2016/06/04/%E5%9C%A8%E7%BA%BF%E6%9B%B4%E6%96%B0Glance%E9%95%9C%E5%83%8F/)

[挂载raw和qcow2格式的KVM硬盘镜像](http://krystism.is-programmer.com/posts/47074.html)

[使用Yum快速更新升级CentOS内核](https://www.sjy.im/toss/use-yum-update-centos-kernel-quickly.html)

[如何挂载一个镜像文件(HOW TO MOUNT AN IMAGE FILE)](http://smilejay.com/2012/08/mount-an-image-file/)

[挂载虚拟机镜像文件里的 LVM 逻辑分区](http://www.vpsee.com/2010/10/mount-lvm-volumes-from-loopback-disk-images/)

[libguestfs ](http://libguestfs.org/)

[kvm虚拟化小结（六）libguestfs-tools](http://www.361way.com/kvm-libguestfs-tools/3175.html)

[libguestfs详解](http://www.hanbaoying.com/2017/02/26/libguestfs.html)

***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***
