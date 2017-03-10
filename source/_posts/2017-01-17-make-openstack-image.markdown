---
layout: post
title:  "openstack 系列--利用Vmware workstation手动制作 openstack centos镜像简记"
date:   2017-01-17 15:17:25
tags: openstack
---

## 目录结构：

[准备工作 ](#A)

[创建虚拟机](#B)

[安装Centos系统](#C)

[配置系统](#D)

[上传镜像](#E)

[自动创建镜像工具](#F)

[参考文章](#G)


tips:在制作镜像的时候最好先知晓镜像所要满足的要求，比如：是否支持磁盘分区，重设根分区大小，密码初始化等等。最常见的需求有如下几种：

- Disk partitions and resize root partition on boot (cloud-init)
- No hard-coded MAC address information
- SSH server running
- Disable firewall
- Access instance using ssh public key (cloud-init)
- Process user data and other metadata (cloud-init)
- Paravirtualized Xen support in Linux kernel (Xen hypervisor only with Linux kernel version < 3.0)

这部分内容可参考官方文档，[Image requirements](http://docs.openstack.org/image-guide/openstack-images.html) 。


<a name="A"></a>

## 准备工作

- 下载Vmware workstation,并安装Centos7,配置好网络。注：刚开始是用的virtualbox,发现不支持nested KVM,Vmware workstation 10+版本以上应该都可以,注意开启workstation的虚拟化设置。
- 宿主机（刚装好的Centos）安装libvirt系列工具:

```bash
yum groupinstall Virtualization "Virtualization Client" 
yum install libvirt
yum install libguestfs-tools
service libvirtd restart 
```

- 从[Centos镜像源](https://www.centos.org/download/mirrors/)下载一个最小的Centos7镜像。本文是以centos为例,其他操作系统类似，详见[Create images manually](http://docs.openstack.org/image-guide/create-images-manually.html#)

```bash
 wget - O http://mirrors.163.com/centos/7.2.1511/isos/x86_64/CentOS-7-x86_64-Minimal-1511.iso
```

<a name="B"></a>

## 创建虚拟机

- 创建一个容量为5G的磁盘,并更改所属者：
```bash
 cd /image
 qemu-img create -f qcow2 centos.qcow2 5G
 chown qemu:qemu /image/centos.qcow2 -R
```

- 创建并启动虚拟机
```bash
 sudo virt-install --virt-type kvm --name centos7 --ram 1024 \  # name 是自己取得
  --disk centos.qcow2,format=qcow2 \      #disk参数为上面创建的磁盘
  --network network=default \             # 默认KVM虚拟机网络为nat
  --graphics vnc,listen=0.0.0.0 --noautoconsole \
  --os-type=linux --os-variant=rhel7 \
  --cdrom=/image/CentOS-7-x86_64-Minimal-1511.iso  #下载的镜像
```

- 用“virsh list” 命令查看虚拟机是否启动，如果没有启动的话用“virsh start $NAME” 启动。
- 执行 “virsh vncdisplay $NAME” ,之后就可以利用vnc-client 访问该虚拟机了。

<a name="C"></a>

## 安装Centos系统

- 利用vnc-viewer 连接 虚拟机，默认是宿主机IP+5900端口。并进行虚拟机的安装。
- 安装完成后，为了能够联外网，以及用宿主机直接SSH 虚拟机，还要进行网络的配置。KVM虚拟机的网络配置方式主要有两种，Bridge模式和NAT 模式，关于两种模式的介绍可以参考这篇文章，[KVM虚拟机网络配置 Bridge方式，NAT方式](http://blog.csdn.net/hzhsan/article/details/44098537/)。我们这里使用默认的NAT模式，但是NAT模式有个弊端就是无法从宿主机SSH虚拟机，这样我们还要做些额外的工作实现该功能。
- 通过端口转发来实现宿主机直连虚拟机，具体参考[这里](http://www.bkjia.com/Linuxjc/877147.html#top)。

```bash
echo 1 >/proc/sys/net/ipv4/ip_forward  # 打开ip转发功能，这种方法是暂时的，可以直接修改/etc/sysctl.conf 文件，增加net.ipv4.ip_forward = 1 达到永久效果，文件修该完毕后，要使用sysctl –p使其生效

iptables -t nat -A POSTROUTING -o br0 -j MASQUERADE  # 开启KVM服务器的IPtables的转发功能

iptables -t nat -A PREROUTING -d 192.168.1.102 -p tcp -m tcp --dport 8022 -j DNAT --to-destination 192.168.122.173:22

iptables -t nat -A POSTROUTING -s 192.168.122.0/255.255.255.0 -d 192.168.122.173 -p tcp -m tcp --dport 22 -j SNAT --to-source 192.168.122.1
```


<a name="D"></a>

## 配置系统

- SSH到虚拟机后就可以开始一系列的配置工作，首先修改yum源为国内源。
- 安装acpid服务，并设置开机自启动（acpid服务是用于hypervisor与虚拟机的交互）。
```bash
yum install -y acpid
systemctl enable acpid
```

- 虚拟机需要打开boot日志功能，并指定console，这样nova console-log才能获取虚拟机启动时的日志。修改配置文件/etc/default/grub，设置GRUB_CMDLINE_LINUX为：
```bash
GRUB_CMDLINE_LINUX="crashkernel=auto console=tty0 console=ttyS0,115200n8"
```
- 手动安装qemu-guest-agent(L 版的动态修改密码需要用到)：

```bash
yum install -y qemu-guest-agent
```

配置qemu-ga，修改/etc/sysconfig/qemu-ga，配置内容为:

```bash
TRANSPORT_METHOD="virtio-serial"
DEVPATH="/dev/virtio-ports/org.qemu.guest_agent.0"
LOGFILE="/var/log/qemu-ga/qemu-ga.log"
PIDFILE="/var/run/qemu-ga.pid"
BLACKLIST_RPC=""
FSFREEZE_HOOK_ENABLE=0
```

- 为了使虚拟机能够和外部的metadata service通信，需要禁用默认的zeroconf route：
```bash
echo "NOZEROCONF=yes" >> /etc/sysconfig/network
```

- 最后安装cloud-init：

```bash
yum install -y cloud-init
```

- 为了实现自动扩容, 需要安装growpart：

```bash
yum update -y
yum install -y epel-release
yum install -y cloud-utils-growpart.x86_64
rpm -qa kernel | sed 's/^kernel-//'  | xargs -I {} dracut -f /boot/initramfs-{}.img {}
```

- 基本完成，虚拟机关机。

- 宿主机上进行清理工作：

```bash
virt-sysprep -d centos7 # 移除mac地址信息
virsh undefine centos7 # 删除虚拟机
```

<a name="E"></a>

## 上传镜像

- 镜像格式转换（ceph只支持raw格式）

```bash
qemu-img convert -f qcow2 -O raw centos.qcow2 centos.raw
```

- 直接用glance上传镜像就不赘述了，这里有个窍门就是如果后端是ceph的话可以利用rbd import 的方式来加快速度，如下。 
- 首先glance create 一条记录，但没有执行镜像上传操作，只是新建一条数据库记录。并记录下image-id。

```bash
glance image-create
```
- 利用ceph import 镜像并执行快照

```bash
rbd --pool=volumes import  centos.raw  --image=$IMAGE_ID --new-format --order 24
rbd --pool=volumes --image=$IMAGE_ID --snap=snap snap create
rbd --pool=volumes --image=$IMAGE_ID --snap=snap snap protect
```

- 完善镜像的必要属性

```bash
glance image-update --name="centos-7.2-64bit" --disk-format=raw --container-format=bare

# 配置qemu-ga，该步骤是必须的，否则libvert启动虚拟机时不会生成qemu-ga配置项，导致虚拟机内部的qemu-ga由于找不到对应的虚拟串行字符设备而启动失败，提示找不到channel
glance image-update --property hw_qemu_guest_agent=yes $IMAGE_ID
```

- 设置镜像的location url
```bash
glance location-add --url rbd://$FS_ROOT/glance_images/$IMAGE_ID/snap $IMAGE_ID#这里的$FS_ROOT 可以通过查看ceph -s 中的cluster.
```


具体参考[手动制作Openstack镜像](http://int32bit.me/2016/05/28/%E6%89%8B%E5%8A%A8%E5%88%B6%E4%BD%9COpenstack%E9%95%9C%E5%83%8F/) 的上传镜像部分。


<a name="F"></a>
## 自动创建镜像工具

openstack 文档提供了几个类似的工具，没有尝试，[Tool support for image creation](http://docs.openstack.org/image-guide/create-images-automatically.html).


<a name="G"></a>

## 参考文章

[openstack doc:Example: CentOS image](http://docs.openstack.org/image-guide/centos-image.html)

[手动制作Openstack镜像](http://int32bit.me/2016/05/28/%E6%89%8B%E5%8A%A8%E5%88%B6%E4%BD%9COpenstack%E9%95%9C%E5%83%8F/)

[制作openstack用的centos6.5镜像](http://yanheven.github.io/centos65-image-create)

[KVM虚拟机网络配置 Bridge方式，NAT方式](http://blog.csdn.net/hzhsan/article/details/44098537/)

[烂泥：KVM使用NAT联网并为VM配置iptables端口转发，kvmiptables](http://www.bkjia.com/Linuxjc/877147.html#top)

[谈谈Openstack的CentOS镜像](http://www.chenshake.com/about-openstack-centos-mirror/)

[Image requirements](http://docs.openstack.org/image-guide/openstack-images.html)

***END***
