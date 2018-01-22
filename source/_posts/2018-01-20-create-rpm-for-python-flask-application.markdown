---
layout: post
title:  "Ops-- 将Flask APP 打包为rpm包"
date:   2018-01-20 15:57:10
tags: 
  - Ops
---


## 概述

为了实现无人值守安装，需要将我们用Python研发的一个WEB做成一个RPM 包，这里简单记录一下。
先简单介绍下需要打包的这个Python应用，是我们为了安装CDH 大数据平台做的一个辅助WEB，使用Flask开发，除了用cherry起的一个WEB 服务外，还有一个celery的worker 进程。 

## 构建思路

- 首先是该Python应用的依赖包，有两种解决方案，一种是全部打包到一个RPM包里，简单粗暴，不容易起冲突，但是不灵活，文件大，每做一次升级比较麻烦，第二种方案是将该应用的依赖包做成RPM包（其实都有现成的），在SPEC文件中注明依赖包，这样在RPM 安装时会自动安装依赖包，这种方案比较灵活，不过需要花时间去找到所有依赖包的RPM 包并放到YUM REPO中。我们采用的是第二种方案。
- 因为要起两个服务，所以要写两个启动文件。
- Python 的setuptools 有一个 python setup.py bdist_rpm 命令用于构建rpm包，不过还是需要自己定制SPEC文件，所以这里直接使用rpmbuild。
- 本次构建的系统环境是centos6.9。


## 具体实践

- 安装rpmbuild
```bash
$ yum install -y rpm-build
```

- 使用普通用户并修改topdir目录
```bash
$ useradd rpmbuilder
$ su - rpmbuilder
$ vim ~/.rpmmacros    # 修改工作目录
  %_topdir        /home/rpmbuilder/rpmbuild

$ mkdir -pv ~/rpmbuild/{BUILD,RPMS,SOURCES,SPECS,SRPMS} 
```
简单介绍下工作目录中的几个目录的作用：
![rpm-dir](http://oeptotikb.bkt.clouddn.com/20180120rpm-dir.jpg)

- 准备源码文件，主要包括三个文件，一个是Flask源码文件全部打包压缩成一个tar.gz包，还有两个启动文件。将这三个文件放到SOURCES目录。启动文件示例如下：
```bash
. /etc/rc.d/init.d/functions

runuser=root
prog=cdhboot-web
worker_num=10
export C_FORCE_ROOT=True
exec="/opt/cdhboot/supervisor/server_cherrypy.py"  # 我们将所有源码文件安装到/opt/cdhboot/目录，后面会提到。
pidfile="/var/run/cdhboot/$prog.pid"

start() {
    [ -f $exec ] || exit 5
    echo -n $"Starting $prog: "
    daemon --user $runuser --pidfile $pidfile "python $exec &>/dev/null & echo \$! > $pidfile"
    retval=$?
    echo
    return $retval
}

stop() {
    echo -n $"Stopping $prog: "
    killproc -p $pidfile $prog
    retval=$?
    echo
    return $retval
}

restart() {
    stop
    start
}

reload() {
    restart
}
force_reload() {
    restart
}

rh_status() {
    status -p $pidfile $prog
}

rh_status_q() {
    rh_status >/dev/null 2>&1
}
case "$1" in
    start)
        rh_status_q && exit 0
        $1
        ;;
    stop)
        rh_status_q || exit 0
        $1
        ;;
    restart)
        $1
        ;;
    reload)
        rh_status_q || exit 7
        $1
        ;;
    force-reload)
        force_reload
        ;;
    status)
        rh_status
    condrestart|try-restart)
        rh_status_q || exit 0
        restart
        ;;
    *)
        echo $"Usage: $0 {start|stop|status|restart|condrestart|try-restart|reload|force-reload}"
        exit 2
esac
exit $?
```
- 编写SPEC文件，SPEC文件有几个部分组成，也代表着rpm打包时的几个步骤。先看下rpm打包的四个步骤：

![rpm-build-step](http://oeptotikb.bkt.clouddn.com/rpmbulld-step-20180122133542.jpg)
对应的SPEC文件示例如下：
```bash
# 第一部分：自定义的变量，通常是包名，版本等信息
%define name cdhboot
%define version 0.1
%define unmangled_version 0.1
%define unmangled_version 0.1
%define release 1

# 第二部分：定义rpm包的信息，三个源文件，依赖包，以及BuildRoot
Summary: cdh boot
Name: %{name}
Version: %{version}
Release: %{release}
Source0: %{name}-%{unmangled_version}.tar.gz 
Source1: cdhboot-worker
Source2: cdhboot-web
License: MIT
Group: Development/Libraries
BuildRoot: /root/rpmbuild/  
Prefix: %{_prefix}
BuildArch: noarch
Vendor: pekingzcc <pekingzcc@gmail.com>
Url: https://github.com/zhangchenchen
Requires: python-celery,python-mongoengine,python-prettytable,python-cherrypy,python-argparse,pytz,python-flask,python-flask-login  # 依赖包，使用yum安装时会先下载安装依赖包
%description
WEB TO BOOT CDH

# 第三部分：准备阶段，%setup是一个宏命令，解压缩包到cdhboot目录，并cd到该目录下。
%prep
%setup -n %{name}-%{unmangled_version} -n cdhboot

# 第四部分：安装之前的操作，添加一个sysadmin用户
%pre
if [ $1 == 1 ];then
    /usr/sbin/useradd  sysadmin -s /sbin/nologin 2> /dev/null
fi

# 第五部分：安装阶段，创建相应目录。并将源文件复制到相应目录。%{__install}是一个宏命令，类似于cp命令。
%install
mkdir -p %{buildroot}/opt/cdhboot
cp -r ./* %{buildroot}/opt/cdhboot/
mkdir -p  %{buildroot}/var/run/cdhboot
mkdir -p  %{buildroot}/var/log/cdhboot
%{__install} -p -D -m 0755 %{SOURCE1} %{buildroot}/etc/rc.d/init.d/cdhboot-worker
%{__install} -p -D -m 0755 %{SOURCE2} %{buildroot}/etc/rc.d/init.d/cdhboot-web

# 第六部分：安转之后的操作，加入开机启动服务
%post
if [ $1 == 1 ];then
    /sbin/chkconfig --add cdhboot-worker
    /sbin/chkconfig cdhboot-worker on
    /sbin/chkconfig --add cdhboot-web
    /sbin/chkconfig cdhboot-web on
fi

# 第七部分：卸载之后的操作，删除sysadmin用户，并停止服务 
%preun
if [ $1 == 0 ];then
        /usr/sbin/userdel  sysadmin 2> /dev/null
        /etc/init.d/cdhboot-worker stop > /dev/null 2>&1
        /etc/init.d/cdhboot-web stop > /dev/null 2>&1
fi

# 第八部分：构建完成后删除临时构建目录内容
%clean
rm -rf $RPM_BUILD_ROOT

# 第九部分：文件部分，但凡上文构建过程中出现的文件或目录，这里都要对这些文件或目录注明属性
%files 
%defattr(-,root,root)
%attr(0755,root,root) /etc/rc.d/init.d/cdhboot-worker
%attr(0755,root,root) /etc/rc.d/init.d/cdhboot-web
/opt
/var/run
/var/log
```

- 测试SPEC文件。为了测试SPEC文件，我们可以分阶段的执行构建命令。
```bash
rpmbuild -bp cdhboot.spec 制作到%prep段
rpmbuild -bc cdhboot.spec 制作到%build段
rpmbuild -bi cdhboot.spec 执行 spec 文件的 "%install" 阶段 (在执行了 %prep 和 %build 阶段之后)。这通常等价于执行了一次 "make install"
rpmbuild -bb cdhboot.spec 制作二进制包
rpmbuild -ba cdhboot.spec 表示既制作二进制包又制作src格式包,即全过程。
```
通过分阶段的构建来测试对应部分的编写正确性，同时，也可以去临时目录BUILDROOT 下查看构建的文件，临时目录BUILDROOT在构建阶段相当于安装时机器的根目录。
构建完成后再RPMS目录下可以看到构建成功的rpm包。

之后可以对rpm包进行签名，就可以放到yum repo中发布了。



## 参考文章

[How to create a rpm for python application](https://stackoverflow.com/questions/42286786/how-to-create-a-rpm-for-python-application)

[使用rpm-build制作nginx的rpm包](http://blog.51cto.com/nmshuishui/1583117)

[记录自己将Python程序打包成rpm包的过程](http://www.voidcn.com/article/p-zvfjwgek-up.html)

[rpmbuild spec文件的编写,以及rpm包的打包](https://wenchao.ren/archives/549)

[How to create an RPM package/zh-cn](https://fedoraproject.org/wiki/How_to_create_an_RPM_package/zh-cn)

***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***