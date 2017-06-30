---
layout: post
title:  "安全系列 --简单加固 ubuntu 服务器"
date:   2016-09-16 15:01:55
tags: security
---


## 目录结构


[前言 ](#A)

[花五分钟的时间加固 ubuntu server ](#B)

[其他](#C)


<a name="A"></a>

## 前言

逛v2ex的时候看到了一篇讨论 [在服务器上做什么加固策略](https://www.v2ex.com/t/306519#reply22)的帖子，想起自己之前看过的一篇文章，[My First 5 Minutes On A Server; Or, Essential Security for Linux Servers](https://plusbryan.com/my-first-5-minutes-on-a-server-or-essential-security-for-linux-servers)，顺手写下这篇文章，权当记录，本篇的绝大部分内容都是出自这篇文章，强烈推荐，如果觉得TLDR,就接着往下翻吧。

安全，在大部分程序员潜意识里总是处于一种low priority的状态（参看此[知乎链接](https://www.zhihu.com/question/20306241)）。比如，我们搭建一个web server，会在第一时间去想用什么去部署，去实现，实现之后可能还会考虑高可用，可扩展等因素。而安全却总是会在最后才出现在我们的脑海里，甚至于直到出现安全事故才去关注。其实，我们拿到一台Linux服务器，只需要花些时间做一些安全方面的考量，做一些简单的措施就可以抵挡很大一部分攻击，提高攻击门槛。

那对于安全方面的措施，我比较推崇上文作者所说：简单有效。因为比较复杂的安全策略实施起来的增量收益可能会低于增加的大量人力物力资源。而且指不定还会引入其他的安全隐患。引用作者的话说：据我的经验来讲，大部分安全事故，要么是没有做足够的安全措施，要么是做了足够的安全措施，但没有更新维护导致出现漏洞。

Simplicity is the heart of good security.

下面就简单说下我们拿到一台服务器（或VPS）后需要做的一些安全加固措施。

<a name="B"></a>

## 花五分钟的时间加固 ubuntu server

### 准备

本文中用的Linux发行版为 14.04。其他实验环境作者没有实践。
首先更改你的 root 密码为强类型密码。然后执行如下命令做一下升级：

```bash
 apt-get update
 apt-get upgrade
```

### 尽量不要使用root用户

新增一个用户，当需要root权限时再用sudo暂时提升权限。新建用户的密码也要是强类型密码。

```bash
  adduser example_user  #新建 用户
  adduser example_user sudo  #给新建用户添加sudo权限
```

### 使用SSH密钥认证登录

利用ssh登录到Linux服务器有很多种方案，比较常用的就是用户名密码登录和SSH密钥认证的方式。
相对用户名密码登陆方式，后者是一种更安全的方案，因为前者容易受到暴力破解的威胁，而后者用暴力破解的方法不太实际。
首先需要在客户端产生公约密钥对。Linux下是如下命令：

```bash
 ssh-keygen -t rsa
```

如果客户端是windows，可以利用putty等客户端实现。

创建完之后，我们会获取到两个文件，id_rsa.pub，和 id_rsa。也就是公钥文件和密钥文件。

接下来将公钥内容复制到服务器.ssh 目录下authorized_keys文件中。

### 关闭SSH 密码服务器登陆
添加如下内容到 /etc/ssh/sshd_config 文件中。
 
```
 PermitRootLogin no
 PasswordAuthentication no
 AllowUsers deploy@(your-ip) deploy@(another-ip-if-any) # 添加允许登录IP
```

重启ssh服务

```bash
 service ssh restart
```

### 添加防火墙

添加防火墙有很多种方法，最简单的就是利用ubuntu 提供的ufw。
```bash
 ufw allow from {your-ip} to any port 22 #限定IP的端口
 ufw allow 80 #打开80端口
 ufw allow 443 #打开443端口
 ufw enable
```

 当然要根据你的实际要求来打开对应的端口。除了ufw，还有iptables,ip6tables,以及最新的nftables等都可以实现防火墙。

### 安装Fail2ban

```bash
 apt-get install fail2ban
```

Fail2ban 会在服务端扫描日志并根据一些策略屏蔽一些有可疑行为的用户（IP）。官方介绍说的比较清楚：

 > Fail2ban scans log files (e.g. /var/log/apache/error_log) and bans IPs that show the malicious signs -- too many password failures, seeking for exploits, etc. Generally Fail2Ban is then used to update firewall rules to reject the IP addresses for a specified amount of time, although any arbitrary other action (e.g. sending an email) could also be configured. Out of the box Fail2Ban comes with filters for various services (apache, courier, ssh, etc).
 Fail2Ban is able to reduce the rate of incorrect authentications attempts however it cannot eliminate the risk that weak authentication presents. Configure services to use only two factor or public/private authentication mechanisms if you really want to protect services.

### 设置安全自动升级

```bash
 apt-get install unattended-upgrades

 vim /etc/apt/apt.conf.d/10periodic
```

更新如下内容：

```bash
 APT::Periodic::Update-Package-Lists "1";
 APT::Periodic::Download-Upgradeable-Packages "1";
 APT::Periodic::AutocleanInterval "7";
 APT::Periodic::Unattended-Upgrade "1";
```
再更新下这个文件/etc/apt/apt.conf.d/50unattended-upgrades

```bash
 Unattended-Upgrade::Allowed-Origins {
        "Ubuntu lucid-security";
//      "Ubuntu lucid-updates";
};
```

### 安装Logwatch

Logwatch 是一个强大的日志解析和分析软件。它被设计用来给出一台服务器上所有的活动报告，可以以命令行的形式输出，或邮件发送。

```bash
 apt-get install logwatch

 vim /etc/cron.daily/00logwatch
```

 添加如下内容：

```bash
 /usr/sbin/logwatch --output mail --mailto test@gmail.com --detail high
```



<a name="C"></a>

## 其他

其实原文在[hacking news](https://news.ycombinator.com/item?id=5316093) 上引发了一系列的讨论。比如有人就提出，我们拿到一台服务器的时候，第一件事不是加固，而是应该先安装配置管理工具（Puppet or Chef），利用配置管理工具才更有条理。其实lz认为还是看实际场景吧，配置管理工具确实方便，尤其是对于一些大型的项目，而对于只搭一个blog的vps，确实没有必要。






## 参考文章

[My First 5 Minutes On A Server; Or, Essential Security for Linux Servers ](https://plusbryan.com/my-first-5-minutes-on-a-server-or-essential-security-for-linux-servers)

[Securing Your Server](https://linode.com/docs/security/securing-your-server)

[Hacking news](https://news.ycombinator.com/item?id=5316093)

[v2ex](https://www.v2ex.com/t/306519#reply22)



***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***

