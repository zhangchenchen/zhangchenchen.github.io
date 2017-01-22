---
layout: post
title:  "python系列--python web 运行与部署"
date:   2016-09-05 14:58:25
tags: python
---

## 目录结构：

在学习一个新东西的时候，我们其实默认是带着问题去学习的。但当我们真正在卷帙繁多的互联网或者书籍里获取信息时。之前的问题往往就淡化了，反而更多的是跟着资料所给的思路去学习。这样有好处也有坏处，好处是我们不用费心去思考，而坏处也是如此，不用思考带来的后果就是这些信息虽然当时我们理解了，但很难在我们的脑海中建立知识图谱，比较容易遗忘。

所以以后我会在写文章的时候先把问题具象化，从自己问题的角度出发去寻找答案。

本篇我们的问题从 *python web 是如何跑起来的？* 开始，一步步探究由此问题引起的其他内容。

[python web 是如何跑起来的？ ](#A)

[探究 WSGI ](#B)

[探究 python web framework](#C)

[探究 python web server](#D)

[常见 python web生产部署环境](#E)





<a name="A"></a>

## python web 是如何跑起来的？

要知道python web 是如何跑起来的，我们首先要知道一个http请求的生命周期。
一个http请求从浏览器发出，在服务端被http server接收，这个http server对请求进行解析，如果是一些图片，文本等静态资源，它就会根据路径去寻找对应的资源并返回。如果是请求的动态资源，比如jsp/php等，它就会把相应的http请求转发给一个web server,由web server对请求进行解析并处理，并最终返回一个 http response。这就是一个http请求的大致周期。
接下来对应到python web中，当web server 接收到一个请求时，它当然是解析请求并把该请求映射到我们写的对应的处理逻辑中。我们的处理逻辑处理完成后，再将结果由web server 转发出去。那web werver 与 我们写的处理逻辑该怎样做才能实现上述效果呢，这就引出了 WSGI。

*以下示例摘自[廖雪峰的官方网站](http://www.liaoxuefeng.com/wiki/001374738125095c955c1e6d8bb493182103fac9270762a000/001386832689740b04430a98f614b6da89da2157ea3efe2000)*

WSGI:Web Server Gateway Interface.
WSGI接口定义非常简单，它只要求Web开发者实现一个函数，就可以响应HTTP请求。

```python
# hello.py

def application(environ, start_response):
    start_response('200 OK', [('Content-Type', 'text/html')])
    return '<h1>Hello, web!</h1>'
```

以上代码中application 函数接收两个参数，environ是一个包含所有HTTP请求信息的dict对象，start_response是一个发送HTTP响应的函数，该函数就是符合WSGI标准的一个HTTP处理函数。调用start_response()就发送了一个http header, http body 就是下文return 的数据。
利用python自带的 web server 加载上面这个application.

```python
# server.py
# 从wsgiref模块导入:
from wsgiref.simple_server import make_server
# 导入我们自己编写的application函数:
from hello import application

# 创建一个服务器，IP地址为空，端口是8000，处理函数是application:
httpd = make_server('', 8000, application)
print "Serving HTTP on port 8000..."
# 开始监听HTTP请求:
httpd.serve_forever()
```

确保以上两个文件在同一个目录下，然后在命令行输入python server.py来启动WSGI服务器。



<a name="B"></a>

## 探究 WSGI 

*以下内容摘自[wsgi和tornado.md](https://gist.github.com/nature-python/8954123)*

WSGI是为python语言定义的web服务器和web应用程序或框架之间的一种简单而实用的接口。wsgi是一个web组件的接口规范，它将web组件分为三类：server，middleware，application。接下来简单介绍下这三个组件：
 
 - wsgi server :可以理解为一个符合wsgi规范的web server，接收request请求，封装一系列环境变量，按照wsgi规范调用注册的wsgi app，最后将response返回给客户端。
 - wsgi application :就是一个普通的callable对象，当有请求到来时，wsgi server会调用这个wsgi app。这个对象接收两个参数，通常为environ,start_response。environ可以理解为环境变量，跟一次请求相关的所有信息都保存在了这个环境变量中，包括服务器信息，客户端信息，请求信息。start_response是一个callback函数，wsgi application通过调用start_response，将response headers/status 返回给wsgi server。此外这个wsgi app会return 一个iterator对象 ，这个iterator就是response body。
 - wsgi middleware :可以简单地理解为对application的封装。通过封装实现一些公用的功能，如下示例用一个简单Dispatcher Middleware，用来实现URL 路由：

```python
from wsgiref.simple_server import make_server

URL_PATTERNS= (
    ('hi/','say_hi'),
    ('hello/','say_hello'),
    )

class Dispatcher(object):

    def _match(self,path):
        path = path.split('/')[1]
        for url,app in URL_PATTERNS:
            if path in url:
                return app

    def __call__(self,environ, start_response):
        path = environ.get('PATH_INFO','/')
        app = self._match(path)
        if app :
            app = globals()[app]
            return app(environ, start_response)
        else:
            start_response("404 NOT FOUND",[('Content-type', 'text/plain')])
            return ["Page dose not exists!"]

def say_hi(environ, start_response):
    start_response("200 OK",[('Content-type', 'text/html')])
    return ["kenshin say hi to you!"]

def say_hello(environ, start_response):
    start_response("200 OK",[('Content-type', 'text/html')])
    return ["kenshin say hello to you!"]

app = Dispatcher()

httpd = make_server('', 8000, app)
print "Serving on port 8000..."
httpd.serve_forever()
```

以上代码就已经有点web framework 的意思啦。由这个实例我们可以看到写一个基于wsgi的python web framework其实就是实现wsgi application部分和wsgi middleware 部分。其实，写一个python web framework 也是一件非常简单的事，见[廖雪峰的官方博客](http://www.liaoxuefeng.com/wiki/001374738125095c955c1e6d8bb493182103fac9270762a000/0014023080708565bc89d6ab886481fb25a16cdc3b773f0000)。下面我们就探究一下业界常用的python web framework。而wsgi server部分我们待会再谈。

<a name="C"></a>

## 探究 python web framework

翻看python 的wiki，我们发现关于python的web framework 以井喷的速度发展，各式各样的框架。截两张图，分别是主流的全栈框架和非全栈框架：

主流的全栈python web框架:
![full-stack](http://7xrnwq.com1.z0.glb.clouddn.com/20160905python-full-stack.jpg)

这里要多说一句tornado框架，其实它不单是一个web框架，还是一个web服务器。作为Web框架，是一个轻量级的Web框架，类似于另一个Python web 框架Web.py，其拥有异步非阻塞IO的处理方式。作为Web服务器，Tornado有较为出色的抗负载能力，官方用nginx反向代理的方式部署Tornado和其它Python web应用框架进行对比，结果最大浏览量超过第二名近40%。见下表：
![tornado 比较](http://7xrnwq.com1.z0.glb.clouddn.com/20160905tornado%20awesome.jpg)

主流的非全栈python web框架：
![non-full-stack](http://7xrnwq.com1.z0.glb.clouddn.com/2016-0905python-non-full-stack.jpg)


<a name="C"></a>

## 探究 python web server

 - Apache + modwsgi，配置安装比较繁琐，用的人越来越少。

 - Gunicorn，运行在UNIX平台上的python wsgi http server。
 
 - uWSGI,实现了uwsgi和WSGI两种协议的Web服务器，负责响应python 的web请求。此处提醒一下，uwsgi是一种uWSGI服务器自有的协议，它跟WSGI毛的关系都没有。关于他们的区别，[看这](http://www.itopers.com:8080/?p=586)
 
 - Tornado,上文已介绍。

附录一张图：
![test](http://7xrnwq.com1.z0.glb.clouddn.com/20160905-test-cgi-glue.png)


<a name="D"></a>

## 常见 python web生产部署环境

![python web deploy](http://7xrnwq.com1.z0.glb.clouddn.com/20160905python-web-deploy.jpg)



## 参考文章

[廖雪峰的官方网站](http://www.liaoxuefeng.com/wiki/001374738125095c955c1e6d8bb493182103fac9270762a000/001386832689740b04430a98f614b6da89da2157ea3efe2000)

[大家都是怎么部署python网站的](https://www.zhihu.com/question/21888077)

[区分wsgi、uWSGI、uwsgi、php-fpm、CGI、FastCGI的概念](http://www.itopers.com:8080/?p=586)

[使用了Gunicorn或者uWSGI,为什么还需要Nginx？](https://www.zhihu.com/question/30560394)

[fcgi vs. gunicorn vs. uWSGI](https://www.peterbe.com/plog/fcgi-vs-gunicorn-vs-uwsgi)

[What is the difference between uwsgi protocol and wsgi protocol?](http://stackoverflow.com/questions/11811434/what-is-the-difference-between-uwsgi-protocol-and-wsgi-protocol)

[理解Python WSGI](http://www.letiantian.me/2015-09-10-understand-python-wsgi/)

[Web Frameworks for Python](https://wiki.python.org/moin/WebFrameworks)

***END***
