---
layout: post
title:  "openstack 系列--制作 openstack 镜像简记"
date:   2017-01-17 15:17:25
tags: openstack
---

## 目录结构：

[准备工作 ](#A)

[创建虚拟机](#B)

[安装Centos系统](#C)

[配置系统](#D)

[上传镜像](#E)

[参考文章](#F)

<a name="A"></a>

## 准备工作

- 下载Vmware workstation,并安装Centos7,配置好网络。注：刚开始是用的virtualbox,发现不支持nested KVM,Vmware workstation 10+版本以上应该都可以。
- 宿主机（刚装好的Centos）安装libvirt系列工具:

```bash
yum groupinstall Virtualization "Virtualization Client" 
yum install libvirt
yum install libguestfs-tools
```

- 从[Centos镜像源](https://www.centos.org/download/mirrors/)下载一个最小的Centos7镜像。

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
- 安装acpid服务，并设置开机自启动。
```bash
yum install -y acpid
systemctl enable acpid
```

- 虚拟机需要打开boot日志功能，并指定console，这样nova console-log才能获取虚拟机启动时的日志。修改配置文件/etc/default/grub，设置GRUB_CMDLINE_LINUX为：
```bash
GRUB_CMDLINE_LINUX="crashkernel=auto console=tty0 console=ttyS0,115200n8"
```
- 手动安装qemu-guest-agent：

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

- 完成，虚拟机关机。

<a name="E"></a>

## 上传镜像

上传镜像就不赘述了，有个窍门就是如果后端是ceph的话可以利用rbd import 的方式来加快速度。具体参考[手动制作Openstack镜像](http://int32bit.me/2016/05/28/%E6%89%8B%E5%8A%A8%E5%88%B6%E4%BD%9COpenstack%E9%95%9C%E5%83%8F/) 的上传镜像部分。

<a name="F"></a>

## 参考文章


[手动制作Openstack镜像](http://int32bit.me/2016/05/28/%E6%89%8B%E5%8A%A8%E5%88%B6%E4%BD%9COpenstack%E9%95%9C%E5%83%8F/)

[制作openstack用的centos6.5镜像](http://yanheven.github.io/centos65-image-create)

[KVM虚拟机网络配置 Bridge方式，NAT方式](http://blog.csdn.net/hzhsan/article/details/44098537/)

[烂泥：KVM使用NAT联网并为VM配置iptables端口转发，kvmiptables](http://www.bkjia.com/Linuxjc/877147.html#top)

[谈谈Openstack的CentOS镜像](http://www.chenshake.com/about-openstack-centos-mirror/)



***END***
