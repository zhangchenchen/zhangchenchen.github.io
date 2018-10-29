---
layout: post
title:  "openstack 系列--Pecan 简介"
date:   2016-08-07 17:06:25
tags: openstack
---


## 简介
Pecan是一个轻量级的python web框架，它专注于HTTP，提供对象路由的寻址方式，没有提供针对session或者数据库的封装功能。尽管是轻量级的，也提供了一些可扩展的特性：

- 对象路由
- REST风格的controller支持
- 可扩展的安全框架
- 可扩展的模板语言支持
- 可扩展的json支持
- 基于python文件的配置

## 安装
[About installation](http://pecan.readthedocs.io/en/latest/installation.html)

## 基于Pecan创建第一个应用
[Quick start](http://pecan.readthedocs.io/en/latest/quick_start.html)

## 对象路由介绍

所谓对象路由就是将Http请求路径map到相应的controller对象的方法。具体操作就是先将URL路径拆分，然后从root controller开始一层层的往下寻找对应的方法，示例如下：

```python
from pecan import expose

class BooksController(object):
    @expose()
    def index(self):
        return "Welcome to book section."

    @expose()
    def bestsellers(self):
        return "We have 5 books in the top 10."

class CatalogController(object):
    @expose()
    def index(self):
        return "Welcome to the catalog."

    books = BooksController()

class RootController(object):
    @expose()
    def index(self):
        return "Welcome to store.example.com!"

    @expose()
    def hours(self):
        return "Open 24/7 on the web."

    catalog = CatalogController()
```

@expose()是一个注解的装饰函数，只有注解了该函数才能进行对象路由，其参数常用于指定返回值，模板。一个request：http://xxxx/catalog/books/bestsellers ,首先会找到RootController，接着找到对应的catalog,也就是CatalogController,再接着找BookController,以此类推。
除此之外，pecan还提供了pecan.route()指定路由匹配，以及 _lookup(), _default(),  _route()函数。详细说明见官方文档。

## Restful 风格的controller

为了兼容 TurboGears2，Pecan也提供了一套rest风格的controller，RestController.

```python
from pecan import expose
from pecan.rest import RestController

from mymodel import Book

class BooksController(RestController):

    @expose()
    def get(self, id):
        book = Book.get(id)
        if not book:
            abort(404)
        return book.title
```

默认的URL mapping 如下：

![url-mapping](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/pecan-restcontroller-url-mapping.jpg)


## 配置文件

当我们新建一个Pecan项目时，默认配置文件如下：

```python
server = {
    'port' : '8080',
    'host' : '0.0.0.0'
}

app = {
    'root' : None,
    'modules' : [],
    'static_root' : 'public',
    'template_path' : ''
}
```

首先是server的配置，就是端口与所在机器的IP.
第二个是app配置，Pecan利用app配置选项将你的应用最终打包成一个wsgi的应用，一个典型的配置如下：

```python
app = {
    'root' : 'project.controllers.root.RootController',
    'modules' : ['project'],
    'static_root'   : '%(confdir)s/public',
    'template_path' : '%(confdir)s/project/templates',
    'debug' : True
}
```

root: 应用的rootcontroller。
modules:就是pecan去找寻你的应用的入口目录，该目录下必须包含一个app.py,该文件必须包含setup_app() 函数。
static_root:静态文件所在根目录。
template_path：模板文件所在目录。
debug：是否开启debug模式。
除了这两个主要配置，还有其他配置，不再赘述。


## Pecan Hooks

利用wsgi中间件访问Pecan内部是一件比较难的事情，Pecan提供了hooks机制来实现类似wsgi中间件的功能。Hooks提供了如下函数让你在request周期内的一些节点上做操作。

- on-route(): 在Pecan将一个请求路由给对应的controller前调用。
- before():路由完成，代码执行前调用。
- after(): 代码执行完后调用
- on_error():一个请求出现异常时调用。

实现一个Pecan Hook

```python
from pecan.hooks import PecanHook

class SimpleHook(PecanHook):

    def before(self, state):
        print "\nabout to enter the controller..."

    def after(self, state):
        print "\nmethod: \t %s" % state.request.method
        print "\nresponse: \t %s" % state.response.status
```

接下来就是将Pecan hook attached 到 controller，有两种，如果是attached到整个项目所有的controller，可以在配置文件中指定，如下：

```python
app = {
    'root' : '...'
    # ...
    'hooks': lambda: [SimpleHook()]
}
```

如果是指定到某一个controller，那么可以用__hooks__做如下操作：

```python
from pecan import expose
from pecan.hooks import HookController
from my_hooks import SimpleHook

class SimpleController(HookController):

    __hooks__ = [SimpleHook()]

    @expose('json')
    def index(self):
        print "DO SOMETHING!"
        return dict()
```

## 参考文章

[pecan doc](http://pecan.readthedocs.io/en/latest/index.html)



***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***
