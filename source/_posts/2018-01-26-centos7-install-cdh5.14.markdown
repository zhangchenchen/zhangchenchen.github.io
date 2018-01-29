---
layout: post
title:  "Bigdata-- Centos7 安装CDH5.14记录"
date:   2018-01-26 16:57:10
tags: 
  - Ops
  - Bigdata
---


## 概述

目标是搭建一个CDH 的测试环境，图方便在网上找了一些中文的搭建博客，结果问题百出，还是回到官网浏览，找到对应的安装文档，花了一点时间搭建，这里记录一下。
系统配置如下：
- centos7.2 minimal,8G内存，100G磁盘(master节点)
- centos7.2 minimal,4G内存，100G磁盘(node节点)
- centos7.2 minimal,4G内存，100G磁盘(node节点)
- centos7.2 minimal,4G内存，100G磁盘(node节点)
搭建的cdh为最新的CDH5.14，感觉master内存还是太少，有点吃力。

## 搭建步骤

CDH的搭建有多种方法，一般来说，都是先搭建Cloudera Manager,然后利用Cloudera Manager搭建CDH，如果是测试环境，可以直接利用Cloudera Manager自动化安装，这种方式使用内嵌的PostgreSQL作为metadata等数据的存储，不适于生产环境。生产环境中一般会使用Mysql或其他独立搭建的数据库（当然要做HA），所以我们会先搭建一个Mysql数据库备用。

### 准备工作

在所有节点上都要执行的工作，包括host设置，设置yum repo，无密钥登录，ntp设置，安装jdk,关闭防火墙等等。

- 修改 hostname，执行命令 hostname NAME,并修改 /etc/hostname文件。
- 修改/etc/hosts文件
```bash
10.10.10.77    cdh1
10.10.10.78    cdh2
10.10.10.79    cdh3
10.10.10.82    cdh4
```
- 关闭防火墙.
```bash
$ systemctl stop firewalld
$ systemctl disable firewalld
``` 
- 所有节点设置无密钥登录,命令大致如下，将最终的authorized_keys再覆盖回所有节点：
```bash
$ ssh-keygen -t rsa
$ cat id_rsa.pub >> authorized_keys
$ chmod 600 authorized_keys
$ scp authorized_keys root@cdh2:~/.ssh/ 
```
- 设置 yum repo.
```bash
$ curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo  # 阿里yum源
$ curl -o /etc/yum.repos.d/cloudera-manager.repo https://archive.cloudera.com/cm5/redhat/7/x86_64/cm/cloudera-manager.repo  # cloudera yum源
```
- ntp设置，安装ntp，编辑crontab定时同步。
```bash
$ yum install ntpd
$ ntpdate time1.aliyun.com
$ crontab e
30 02 * * *  ntpdate time1.aliyun.com 
```
- 安装jdk.
```bash
 yum install oracle-j2sdk1.7 -y
```
- 关闭SElinux,修改/etc/selinux/config为disabled，重启生效。
```bash
# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#       enforcing - SELinux security policy is enforced.
#       permissive - SELinux prints warnings instead of enforcing.
#       disabled - SELinux is fully disabled.
SELINUX=disabled
# SELINUXTYPE= type of policy in use. Possible values are:
#       targeted - Only targeted network daemons are protected.
#       strict - Full SELinux protection.
SELINUXTYPE=targeted
```
### master节点安装与配置Mysql 
- 安装mysql.
```bash
$ wget http://repo.mysql.com/mysql-community-release-el7-5.noarch.rpm
$ sudo rpm -ivh mysql-community-release-el7-5.noarch.rpm
$ yum update
$ sudo yum install mysql-server
$ sudo systemctl start mysqld
```
- 删除/var/lib/mysql/ib_logfile0 和 /var/lib/mysql/ib_logfile1文件，并配置/etc/my.conf如下：
```bash
[mysqld]
transaction-isolation = READ-COMMITTED
# Disabling symbolic-links is recommended to prevent assorted security risks;
# to do so, uncomment this line:
# symbolic-links = 0

key_buffer_size = 32M
max_allowed_packet = 32M
thread_stack = 256K
thread_cache_size = 64
query_cache_limit = 8M
query_cache_size = 64M
query_cache_type = 1

max_connections = 550
#expire_logs_days = 10
#max_binlog_size = 100M

#log_bin should be on a disk with enough free space. Replace '/var/lib/mysql/mysql_binary_log' with an appropriate path for your system
#and chown the specified folder to the mysql user.
log_bin=/var/lib/mysql/mysql_binary_log

# For MySQL version 5.1.8 or later. For older versions, reference MySQL documentation for configuration help.
binlog_format = mixed

read_buffer_size = 2M
read_rnd_buffer_size = 16M
sort_buffer_size = 8M
join_buffer_size = 8M

# InnoDB settings
innodb_file_per_table = 1
innodb_flush_log_at_trx_commit  = 2
innodb_log_buffer_size = 64M
innodb_buffer_pool_size = 4G
innodb_thread_concurrency = 8
innodb_flush_method = O_DIRECT
innodb_log_file_size = 512M

[mysqld_safe]
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid

sql_mode=STRICT_ALL_TABLES
```
- 安装 MySQL JDBC Driver从[该页面](http://www.mysql.com/downloads/connector/j/5.1.html)下载对应tar包，解压。
```bash
$ tar zxvf mysql-connector-java-5.1.31.tar.gz
$ mkdir -p /usr/share/java/
$ cp mysql-connector-java-5.1.31/mysql-connector-java-5.1.31-bin.jar /usr/share/java/mysql-connector-java.jar
```
- 创建对应的数据库。
```bash
mysql> create database database DEFAULT CHARACTER SET utf8;
Query OK, 1 row affected (0.00 sec)

mysql> grant all on database.* TO 'user'@'%' IDENTIFIED BY 'password';
Query OK, 0 rows affected (0.00 sec)
```
数据库的名字，用户名等可以参考除了以下列出的，还需创建oozie的数据库以及用户名密码.
![mysql](http://oeptotikb.bkt.clouddn.com/20180129200028-mysql-role.jpg)

### Cloudera Manager 安装

- 在master节点安装Cloudera Manager Server并启动。
```bash
$ yum install cloudera-manager-daemons cloudera-manager-server
$ systemctl start cloudera-scm-server
```
- 在master和node节点安装Cloudera Manager Agent。修改 /etc/cloudera-scm-agent/config.ini 中的server_host为master的IP。
```bash
$ yum install cloudera-manager-agent cloudera-manager-daemons
$ systemctl start cloudera-scm-agent
```

### CDH 安装

进入 Cloudera Manager的console，http://Server host:7180,登录后便可以进入CDH的安装部署了。

该过程没有截图，且因为web界面看着比较直白不在赘述。


## 参考文章

[Installation Path B - Installation Using Cloudera Manager Parcels or Packages](https://www.cloudera.com/documentation/enterprise/latest/topics/cm_ig_install_path_b.html#id_z2h_pnm_25)

[Connect Hue to MySQL or MariaDB](https://www.cloudera.com/documentation/enterprise/latest/topics/hue_dbs_mysql.html#concept_tq4_tbt_zw)

[Troubleshooting Installation and Upgrade Problems](https://www.cloudera.com/documentation/enterprise/latest/topics/cm_ig_troubleshooting.html#cmig_topic_19)

[离线部署 Cloudera Manager 5 和 CDH 5.12.1 及使用 CDH 部署 Hadoop 集群服务](https://segmentfault.com/a/1190000011341408)

[Centos 7安装CDH 5.13.0总结](https://www.cmgine.com/archives/19107.html)


***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***