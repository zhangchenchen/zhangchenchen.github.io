---
layout: post
title:  "Ops-- ntp设置记录"
date:   2017-11-03 10:03:10
tags: 
  - Ops
---


## 概述

简单记录下ntp 的配置过程。有三台机器，系统都是Centos7,如下：
- 172.16.21.250,作为ntpclient从公网阿里的ntpserver获取时间，同时作为ntpserver为内网的client提供时间服务。
- 172.16.21.251，作为内网ntpclient。
- 172.16.21.252，作为内网ntpclient

## ntp server部署配置

- 安装rpm包：yum install ntp -y
- 建议关闭firewalld,如果开启了iptables，要放开udp 的123端口。添加
 ```bash
 -AINPUT -p udp -m state --state NEW -m udp --dport 123 -j ACCEPT
 ```
- 若开启了chronyd服务，关闭并关闭开机自启,chronyd为centos7默认时钟同步工具，会出现冲突。
```bash
systemctl stop chronyd & systemctl disable chronyd
```
- 修改配置文件。主要是两个地方，一个是连接它的客户端限制，一个是它连接的服务端设置。
```bash
# Permit all access over the loopback interface.  This could
# be tightened as well, but to do so would effect some of
# the administrative functions.
restrict 172.16.21.0 mask 255.255.255.0 nomodify # 设定客户端的IP限制，且没有更改服务端时间的权限
restrict 127.0.0.1 # 这里添加本机host，因为执行ntpq -p 需要loopback
.........
# 添加阿里的ntpserver
server ntp1.aliyun.com prefer
server ntp2.aliyun.com
server ntp3.aliyun.com
server ntp4.aliyun.com
server ntp5.aliyun.com
..........
```
- 开启ntpd，并设为开机自启。

```bash
systemctl start ntpd & systemctl enable ntpd
```

- 查看服务端是否正常工作：

```bash
$ ntpq -p
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
*time5.aliyun.co 10.137.38.86     2 u    5  128  377    3.265    1.952   0.763
+120.25.115.19   10.137.38.86     2 u  121  128  377   36.172    2.273   0.583
-120.25.115.20   10.137.38.86     2 u  126  128  377   38.508    6.644   0.581
+time4.aliyun.co 10.137.38.86     2 u   16  128  377   37.158    1.693   0.582
```

## ntp client部署配置

- 安装rpm包：yum install ntp -y
- 测试是否安装成功,并获取当前时间，通过hwclock–r写入bios时间
```bash
$ ntpdate 172.16.21.250
3 Nov 10:32:13 ntpdate[8008]: adjust time server 172.16.21.250 offset -0.076117 sec
$ date; hwclock -r
```
- 加入crontab，设置自动同步（每隔三个小时更新一次时间）

```bash
$ crontab -e 0 */1 * * * /usr/sbin/ntpdate 172.16.21.250;/sbin/hwclock -w >/dev/null 2>&1
$ crontab –l
0 */1 * * * /usr/sbin/ntpdate 172.16.21.250;/sbin/hwclock -w >/dev/null 2>&1
```
- 如果时区不对，可以这样修改：
```bash
$ timedatectl set-timezone Asia/Shanghai  # 修改为北京时间
$ timedatectl status #查看是否修改成功
```


## 参考文章


[Centos7.1 for NTP服务器配置 ](http://bigtrash.blog.51cto.com/8966424/1826481)

[ntpq fails with "timed out, nothing received" and "Request timed out" errors](https://access.redhat.com/solutions/256813)


***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***