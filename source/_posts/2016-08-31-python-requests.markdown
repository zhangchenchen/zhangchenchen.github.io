---
layout: post
title:  "python库系列--requests 与 刷票"
date:   2016-08-31 10:24:25
tags: python
---

## 目录结构：

[关于刷票的若干思考 ](#A)

[requests 引出 ](#B)

[requests 基本用法](#C)

[requests 高级用法](#D)





<a name="A"></a>

## 关于刷票的若干思考

前两天同学让我帮他去一个网站帮他刷票，就顺手写了一个小程序，整理一下关于刷票的若干思考：
其实应付普通的投票机制非常简单，只要懂点http的相关知识就可以搞定，原理就是构造相应的HTTP请求。一般来说，为了防止刷票，都会有以下几种防刷票策略：
 
 - 限制IP : 相应的对策就是伪造IP。要想伪造IP的话首先要知道后端服务器程序是怎么获取IP的。至少有三个HTTP HEAD可以获取用户端IP，REMOTE_ADDR、HTTP_VIA、HTTP_FORWARDED_FOR 。REMOTE_ADDR是WEB服务器获取的用户IP值，也就是最终的外网IP，但它的重复值太多，所以不能作为唯一判断标志。 HTTP_VIA是倒数第二个代理服务器/网关的IP。 HTTP_FORWARDED_FOR是所有网关和你自己的IP列表。大部分服务端程序都是用HTTP_FORWARDED_FOR来获取IP。这样我们就可以通过修改HTTP_FORWARDED_FOR来达到伪造IP的目的。

 - 限制网页来源 : 修改 http header中的refer。

 - 限制 user-agent : 修改 http header中的 user-agent。

 - 限制cookie : 修改 http header中的 cookie

 - 限制登录后才可以投票 :模拟登录

 - 验证码验证 : 简单的验证码可以通过OCR识别技术，增加干扰策略的验证码就比较复杂了，不予讨论。

当然，其实防刷票机制不只是上面简单的几种，一般来说比较安全的网站都会制定一些策略，比如IP可疑就增加验证码难度，设定黑白名单等。


<a name="B"></a>

## requests 引出

因为写上面的那个小程序用的是requests库。就简单的学习了一下这个库。
python有自己的标准网络库，urllib,urllib2。但这两个库用着都不是特别的顺畅，requests 库的出现让人们眼前一亮，官网的declare:HTTP for Humans 。强调人性化。


<a name="C"></a>

## requests 基本用法

requests包要完成的内容无非就是发送和接收http请求。接下来我们就从发送请求和接收请求两方面讨论下它的基本使用。

 - 发送请求 ：首先最简单的就是常用的get，post等发送HTTP请求类型，requests是这样发送的：

```python
 
 import requests

 r = requests.get('https://github.com/timeline.json')
 r = requests.post("http://httpbin.org/post")
 r = requests.put("http://httpbin.org/put")
 r = requests.delete("http://httpbin.org/delete")
 r = requests.head("http://httpbin.org/get")
 r = requests.options("http://httpbin.org/get")

```

 对于 get 请求，我们可以通过添加params添加关键字参数：

```python
 import requests

 param = {'key1': 'value1', 'key2': 'value2'}
 r = requests.get("http://httpbin.org/get", params=param)

```

 对于post请求，我们可以通过data参数来模拟提交表单。

```python
 import requests

 param = {'key1': 'value1', 'key2': 'value2'}
 r = requests.get("http://httpbin.org/get", data=param)

```

 还可以通过 headers参数定制我们发送的请求：

```python
 import requests

 my_headers = {
    'User-Agent' : 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_9_5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/48.0.2564.116 Safari/537.36',
    'Accept' : 'text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8',
    'Accept-Encoding' : 'gzip',
    'Accept-Language' : 'zh-CN,zh;q=0.8,en;q=0.6,zh-TW;q=0.4'
}
 r = requests.get("http://httpbin.org/get", headers=my_headers)

```

 除此之外，还有timeout，cookies参数来设置响应时间，cookie。

 - 相应内容 ： 任何时候调用 requests.*() 你都在做两件主要的事情。其一，你在构建一个 Request 对象， 该对象将被发送到某个服务器请求或查询一些资源。其二，一旦requests得到一个从服务器返回的响应就会产生一个 Response 对象。该响应对象包含服务器返回的所有信息， 也包含你原来创建的 Request 对象。
 
```python
 import requests

 r = requests.get('https://github.com/timeline.json')
 
 print(r.text) # 字符形式，可以自己设置编码

 print(r.content) # 二进制形式

 print(r.json()) # json形式,requests 中有一个内置的 JSON 解码器,处理 JSON 数据。

 print(r.raw()) # 原始套接字形式
```


<a name="D"></a>

## 高级用法

 - 会话对象：http是无状态的，为了能够获取用户登录状态，一般服务器会用一个sessionId放在cookie中。用户发送的请求也包含这个sessionId，这些工作由浏览器来做。而requests也实现了类似功能。

```python
 import requests
 s = requests.Session()

 s.get('http://httpbin.org/cookies/set/sessioncookie/123456789')
 r = s.get("http://httpbin.org/cookies")

 print(r.text)
  #'{"cookies": {"sessioncookie": "123456789"}}'

```

 - SSL 证书验证
 Requests 可以为 HTTPS 请求验证 SSL 证书，就像 web 浏览器一样。要想检查某个主机的 SSL 证书，可以使用 verify 参数:

```python
 import requests

 requests.get('https://github.com', verify=True)

```

 - 代理 
 利用proxies 参数来配置请求 

```python
 import requests

 proxies = {
   "http": "http://10.10.1.10:3128",
   "https": "http://10.10.1.10:1080",
 }

 requests.get("http://example.org", proxies=proxies)

```



## 参考文章

[requests quickstart](http://docs.python-requests.org/zh_CN/latest/user/quickstart.html)

[requests advance ](http://docs.python-requests.org/zh_CN/latest/user/advanced.html)

[Python HTTP 库：requests 快速入门](http://liam0205.me/2016/02/27/The-requests-library-in-Python/)


***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***
