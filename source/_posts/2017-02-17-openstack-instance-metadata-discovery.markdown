---
layout: post
title:  "openstack-- openstack instance metadata 服务机制探索"
date:   2017-02-17 17:54:25
tags: openstack
---



## 引出

在调试 cloud-init的时候发现在虚拟机内执行“ curl 169.254.169.254” 的时候出现“host not reachable”错误。在对 openstack instance（确切的说应该是server,因为还没有正式生成云主机） metadata 服务机制探索了一番之后，发现了错误所在，在分析该问题之前先记录下metadata 服务的原理，最后再记录下问题的解决过程。


## openstack instance metadata 服务机制

广义上的metadata包含很多，不光 instance 有metadata属性，image,host,aggregate等都有。在openstack中，因为instance利用metadata 属性可以做很多事情，研究的价值更高一些，所以这里只讨论 instance 的metadata。(注：这里的metadata 服务 泛指 提供userdata && metadata 的服务，因为userdata与metadata只是 内容以及 虚拟机获取之后处理方式不同，虚拟机获取方式是一样的)。关于 userdata 与 metadata，这里直接引用[OpenStack 的 metadata 服务机制](http://www.ibm.com/developerworks/cn/cloud/library/1509_liukg_openstackmeta/)中的一段：

 > 在创建虚拟机的时候，用户往往需要对虚拟机进行一些配置，比如：开启一些服务、安装某些包、添加 SSH 秘钥、配置 hostname 等等。在 OpenStack 中，这些配置信息被分成两类：metadata 和 user data。Metadata 主要包括虚拟机自身的一些常用属性，如 hostname、网络配置信息、SSH 登陆秘钥等，主要的形式为键值对。而 user data 主要包括一些命令、脚本等。User data 通过文件传递，并支持多种文件格式，包括 gzip 压缩文件、shell 脚本、cloud-init 配置文件等。这些信息被获取到后被cloudinit 利用并实现各自目的。虽然 metadata 和 user data 并不相同，但是 OpenStack 向虚拟机提供这两种信息的机制是一致的，只是虚拟机在获取到信息后，对两者的处理方式不同罢了
 > 

openstack 中，虚拟机获取metadata 的信息有两种，即config drive 和 metadata restful 服务（即EC2方式）。下面分别记录下两种方式的原理：


### config drive 的方式

所谓 config drive的方式就是将metadata信息写入虚拟机的一个特殊配置设备中，虚拟机启动的时候会挂载并读取metadata信息。要想实现这个功能，还需要宿主机和镜像满足一些条件：
 
 1. 宿主机满足条件：
       - 虚拟化方式为以下几种：libvirt,xen,hyper-v,vmware.
       - 对于 libvirt, xen, vmware的虚拟化方式，需要安装genisoimage.并且需要设置mkisofs为 程序的安装目录，如果是跟nova-compute安装在一个目录内就不用了。
       - 如果是 hyper-v的虚拟化方式，需要用mkisofs的命令行指定mkisofs.exe 程序的绝对安装路径，且在hyperv的配置文件中设置qemu_img_cmd 为qemu-img 的命令行安装路径。
 2. 镜像满足条件：
       - 安装了cloud-init，且最好是0.7.1+ 的版本。如果是没有安装的话，就需要自己去定制脚本了。
       - 如果使用xen 虚拟化，需要配置 xenapi_disable_agent 为true。

在openstack中，在使用命令行创建虚拟机的时候指定 --config-drive true 即可使用config drive。如下实例（也可以用 nova command）：

```bash
openstack server create --config-drive true --image my-image-name \
  --flavor 1 --key-name mykey --user-data ./my-user-data.txt \
  --file /etc/network/interfaces=/home/myuser/instance-interfaces \
  --file known_hosts=/home/myuser/.ssh/known_hosts \
  --property role=webservers --property essential=false MYINSTANCE
```

也可以在/etc/nova/nova.conf配置文件中指定 force_config_drive = true ，那么默认就使用config drive。

如果镜像操作系统支持通过标签访问磁盘的话，可以使用如下命令查看config drive中的内容：
```bash
mkdir -p /mnt/config
mount /dev/disk/by-label/config-2 /mnt/config
cd /mnt/config & ls -l
```
进入该目录后，看到的内容大致如下：
```bash
ec2/2009-04-04/meta-data.json
ec2/2009-04-04/user-data
ec2/latest/meta-data.json
ec2/latest/user-data
openstack/2012-08-10/meta_data.json
openstack/2012-08-10/user_data
openstack/content
openstack/content/0000
openstack/content/0001
openstack/latest/meta_data.json
openstack/latest/user_data
```
可以看出，config drive支持 openstack以及EC2的方式获取数据，官方手册建议使用opensatck的方式（即读取opensatck目录里的内容），因为EC2的目录以后可能会弃用。而且最好使用最近的版本目录读取。


###  metadata restful 服务的方式

就是我们在虚拟机通过169.254.169.254的方式来获取metadata的方式。由以下三个组件来完成这项工作：

 - Nova-api-metadata：运行在计算节点，启动restful服务，真正负责处理虚拟机发送来的rest 请求。从请求头中获取租户，虚拟机ID，再去数据库中查询相应的metadata信息并返回结果。
 - Neutron-metadata-agent：运行在网络节点，负责将接收到的获取 metadata 的请求转发给 nova-api-metadata。
 - Neutron-ns-metadata-proxy：由于虚拟机获取 metadata 的请求都是以路由和 DHCP 服务器作为网络出口，所以需要通过 neutron-ns-metadata-proxy 联通不同的网络命名空间，将请求在网络命名空间之间转发。

流程大概如下：

![metadata](http://www.ibm.com/developerworks/cn/cloud/library/1509_liukg_openstackmeta/index2894.png)


具体看下路由的过程，如果虚拟机所在的subnet连接在router上，该请求会先通过router来发送请求，如果没有router相连的话会通过对应dns 的网络空间进行传递，示例可以查看[OpenStack 的 metadata 服务机制](http://www.ibm.com/developerworks/cn/cloud/library/1509_liukg_openstackmeta/) 文章末。







## 问题解决

了解了metadata rest的原理后，解决上述问题的思路就比较明确了，首先看下虚拟机确是使用restful 服务的方式而不是config drive的方式，从网络拓扑看，虚拟机所在的子网是连接着router的，但是查看虚拟机的路由表，却发现169.152.169.254发送到了dns的网络空间，而不是router的网络空间，然后再去查看对应dns的网络空间，发现并没有监听 80 端口的 neutron-ns-metadata-proxy 服务，而在router的网络空间却有对应的路由规则。
大致猜测虚拟机boot的时候没有发现对应子网有router相连，所以该虚拟机boot后会将169.152.169.254路由到对应dns，但其实该子网是有router相连的，所以neutron还是将监听的路由规则放到了router上。与router相关的服务就是neytron-l3-agent,重启该服务后，一切正常。





## 参考文章

[OpenStack 的 metadata 服务机制](http://www.ibm.com/developerworks/cn/cloud/library/1509_liukg_openstackmeta/)

[Store metadata on a configuration drive](https://docs.openstack.org/user-guide/cli-config-drive.html)

[metadata-service](https://docs.openstack.org/admin-guide/compute-networking-nova.html#metadata-service)



***END***
