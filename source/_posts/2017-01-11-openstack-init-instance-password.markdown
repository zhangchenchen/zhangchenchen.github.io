---
layout: post
title:  "openstack系列--初始化虚拟机实例密码几种方法"
date:   2017-01-13 14:57:25
tags: openstack
---

## 目录结构：

[引出](#A)

[关于cloud init](#B)


[openstack 初始化虚拟机密码几种方式](#B)


<a name="A"></a>

## 引出

- 最初的问题是这样的：公司的云平台没有提供虚拟机密码重置的功能，需要加上，但我看到创建虚拟机的时候是可以设置密码的，于是想将重置密码的功能重用这部分，结果发现问题不是这么简单。
- 关于虚拟机重置密码放在[下一篇](http://zhangchenchen.github.io/2017/01/19/openstack-reset-instance-password/)，今天主要写写初始化密码的几种方式。
- 其实虚拟机初始化设置密码，跟虚拟机初始化设置网络，主机名是一样的道理，都是通过cloud-init来实现。
- 先看下cloud-init。


<a name="B"></a>

## 关于cloud init

- Cloud-Init 是一个用来自动配置虚拟机的初始设置（如主机名，网卡和密钥）的工具（支持Linux,对于Windows系列有一个类似的工具，[cloudbase-init](https://github.com/cloudbase/cloudbase-init)）。它可以在使用模板部署虚拟机时使用，从而达到避免网络冲突的目的。在使用这个工具前，cloud-init 软件包必须在虚拟机上被安装。安装后，Cloud-Init 服务会在系统启动时搜索如何配置系统的信息。您可以使用只运行一次窗口来提供只需要配置一次的设置信息；或在新建虚拟机、编辑虚拟机和编辑模板窗口中输入虚拟机每次启动都需要的配置信息。（以上内容直接引用自red hat 官网）
- cloud-init 的配置数据有两种：
    1. userdata:文件的形式，常见的如yaml文件，shell scripts，cloud config file。（cloud-init会自动识别以 "#!" 或 "#cloud-config" 开头的文件。）
    2. metedata:键值对的形式，常见的如：server name, instance id, display name and other cloud specific details.
- cloud-init 的数据源：即cloud-init会从以下几种方式获取userdata或metedata：
    1. EC2方式：顾名思义，就是AWS的那一套，其实本质是创建一个http server,虚拟机通过这个http server获取到instance的userdata和metadata。这个http server的IP 通常为169.254.169.254。
    2. Config Drive方式：主要是openstack提供的一种方式，本质是把data写入一个特殊的配置设备中，然后在虚拟机启动时，自动挂载并读取 metadata 信息。
    3. Alt cloud：通常用于 RHEVm 和 vSphere中获取userdata。
    4. vSphere 用于为了向VMWare的vSphere虚拟机中注入userdata。
    5. No Cloud 允许用户在没有网络的情况下提供user-data和medatata给instances



<a name="C"></a>

## openstack 初始化虚拟机密码几种方式

因为目前大部分私有云平台基本是KVM虚拟化，后台存储为ceph，所以只选适合这种架构的方案，早期的[inject-password](http://niusmallnan.com/_build/html/_templates/openstack/inject_passwd.html#inject),貌似只支持QCOW2，不支持RAW镜像(没有验证，但在ceph的文档里确实把这个功能的相关配置默认为false,而且推荐使用cloud-init)以及只支持XEN的clear-password ， root-password等API不再考虑。

1. 利用user-data注入：我司的云平台就是使用这种方式，用法很简单

    ![user-data](http://7xrnwq.com1.z0.glb.clouddn.com/2017-1-13-cloud-init-test.png)

    cloud-init数据源是采用最广泛的EC2模式，EC2的restAPI相关知识[check this](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html#instancedata-add-user-data)。关于metadata  server的配置不在描述，主要是Nova，和Neutron 的相关配置（默认配置即可）。
    登录到对应的虚拟机验证下：

    ![metadata](http://7xrnwq.com1.z0.glb.clouddn.com/2017-01-13-metadata.png)

    关于metadata和userdata的服务机制,参考[OpenStack 的 metadata 服务机制](http://www.ibm.com/developerworks/cn/cloud/library/1509_liukg_openstackmeta/)

2. 修改cloud-init的方式：注意在执行命令nova boot的时候会生成一个adminPass的字段。当然，如果你想用自己的密码，也可以在meta中指定，然后修改下代码即可，参考[这里](https://segmentfault.com/a/1190000002878435) 。
     
     ![adminPass](http://7xrnwq.com1.z0.glb.clouddn.com/2017-01-14-adminPass.png) 

    我们可以获取该字段，然后交给cloud-init进行密码的设置。前文所说，虚机在通过cloud-init获取元数据时可以使用api-metadata、ConfigDrive等方式，但是因为只有ConfigDrive方式才会把adminPass字段传递给虚机，所以这里我们只能用config drive的方式。关于config drive的使用方式，[check this](http://www.voidcn.com/blog/wuyongpeng0912/article/p-6099231.html) 。而且，为了能够使cloud-init获取到该字段，需要加个patch:

 ```python
diff --git a/cloudinit/config/cc_set_passwords.py b/cloudinit/config/cc_set_passwords.py
index 4ca85e2..5b5cae4 100644
--- a/cloudinit/config/cc_set_passwords.py
+++ b/cloudinit/config/cc_set_passwords.py
@@ -44,6 +44,12 @@ def handle(_name, cfg, cloud, log, args):
     else:
        password = util.get_cfg_option_str(cfg, "password", None)

-    # use the admin_pass available in the ConfigDrive
-    if not password:
-        metadata = cloud.datasource.metadata
-        if metadata and 'admin_pass' in metadata:
-            password = metadata['admin_pass']
+
     expire = True
     pw_auth = "no"
     change_pwauth = False
@@ -59,6 +65,8 @@ def handle(_name, cfg, cloud, log, args):
         (user, _user_config) = ds.extract_default(users)
         if user:
             plist = "%s:%s" % (user, password)
-            #add change root password
-            plist = plist + "\nroot:%s" % password
         else:
             log.warn("No default or defined user to change password for.")
```
    同时cloud.cfg中也要添加:
    chpasswd: { expire: False }
    这样保证修改的密码不是过期的，否则vm启动后输入密码，系统让你重新修改才能进入。

3, 利用L版的新特性set-admin-password：这种方式本身要求的条件有点高，没有去尝试，详见[虚拟机系统密码的修改方案](http://niusmallnan.com/_build/html/_templates/openstack/inject_passwd.html#id2)





## 参考文章

[How to reset password of openstack instance using KVM and libvirt?](http://stackoverflow.com/questions/24864186/how-to-reset-password-of-openstack-instance-using-kvm-and-libvirt)

[Cloud Init and Config Drive](https://github.com/jriguera/ansible-ironic-standalone/wiki/Cloud-Init-and-Config-Drive)

[Cloud-init初探 和 OpenStack应用 ](http://blog.sina.com.cn/s/blog_959491260101m2cx.html)

[openstack 中 metadata 和 userdata 的配置和使用](http://www.itdadao.com/articles/c15a642783p0.html)

[虚拟机系统密码的修改方案](http://niusmallnan.com/_build/html/_templates/openstack/inject_passwd.html#id2)

[设置虚拟机密码](http://kiwik.github.io/openstack/2016/01/30/%E8%AE%BE%E7%BD%AE%E8%99%9A%E6%8B%9F%E6%9C%BA%E5%AF%86%E7%A0%81/)

[Cloud config examples](http://cloudinit.readthedocs.io/en/latest/topics/examples.html)



***END***
