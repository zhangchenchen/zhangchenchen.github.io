---
layout: post
title:  "openstack 系列--nova-api 的paste&route的简单抽象"
date:   2017-01-12 15:17:25
tags: openstack
---

## 目录结构：

[由Python的调用实例想到的 ](#A)

[nova-api 的paster&route 模型](#B)


<a name="A"></a>

## 由Python的调用实例想到的

- 公司的云平台有两个自己写的服务，highland和rainflow。也没有开发文档，只能自己去看代码琢磨一下，从paste配置文件看下最终调用的route app，是一个factory方法，我记得factory方法应该返回一个WSGI APP,但是看下源码是这样的：

```python
    @classmethod
    def factory(cls, global_config, **local_config):
        """Simple paste factory, :class:`rainflow.wsgi.Router` doesn't have"""
        return cls()
```

是一个类方法返回这个类，我就有点懵了，这样是执行什么操作呢，难道是再创建一个实例。而且nova-api也是这么干的，证明不是错误。后来搜了一下资料，再试验了一下大致知道怎么回事了。
- 看下下面这个示例：

```python
#!/usr/bin/env python
class demo:
    @classmethod
    def iam_classmethod(cls):
        print("classmethod")
        return cls()

    def __init__(self):
        print("i am init")

    def __call__(self):
        print"i am calling"

test=demo.iam_classmethod()
test()
```

运行结果为：

```
classmethod
i am init
i am calling
```

iam_classmethod方法会返回一个类实例，执行这个类实例就是调用__call__() 方法。
同理，上面那个代码其实就是调用__call__方法，我们可以在对应的父类找到。

<a name="B"></a>

## nova-api 的paster&route 模型

- nova-api的路由模型其实不难，只不过层层封装，比较复杂，在网上找到一个比较好理解的示例。
- 首先要明白路由的生成过程跟路由的寻址过程是两条线，当我们启动nova-api的时候，路由的map就已经生成了，当我们发送rest请求时，就会根据路由的map去找对应的application以及controller。所以，重点是看下路由的生成过程。

![paster&route](http://7xrnwq.com1.z0.glb.clouddn.com/2017-01-12-openstack-paste-route.png)

- 看下一个简单的代码实现（代码来自网上，没有经过验证，只是为了说明）：

```python
from __future__ import print_function
from routes import Mapper
import webob.dec
import webob.exc
import routes.middleware
import testtools

class MyController(object):
    def getlist(self, mykey):
        print("step 4: MyController's getlist(self, mykey) is invoked")
        return "getlist(), mykey=" + mykey

class MyApplication(object):
    """Test application to call from router."""

    def __init__(self, controller):
        self._controller = controller
        
    def __call__(self, environ, start_response):
        print("step 3: MyApplication is invoked")
        
        action_args = environ['wsgiorg.routing_args'][1].copy()
        try:
            del action_args['controller']
        except KeyError:
            pass

        try:
            del action_args['format']
        except KeyError:
            pass
        
        action = action_args.pop('action', None)
        controller_method = getattr(self._controller, action)
        result = controller_method(**action_args)
        
        start_response('200 OK', [('Content-Type', 'text/plain')])
        return [result]

class MyRouter(object):
    """Test router."""

    def __init__(self):
        route_name = "dummy_route"
        route_path = "/dummies"
        
        my_application = MyApplication(MyController()) 
        
        self.mapper = Mapper()
        self.mapper.connect(route_name, route_path,
                        controller=my_application,
                        action="getlist",
                        mykey="myvalue",
                        conditions={"method": ['GET']})
        
        
        self._router = routes.middleware.RoutesMiddleware(self._dispatch,
                                                          self.mapper)

    @webob.dec.wsgify(RequestClass=webob.Request)
    def __call__(self, req):
        """Route the incoming request to a controller based on self.map.

        If no match, return a 404.

        """
        print("step 1: MyRouter is invoked")
        return self._router

    @staticmethod
    @webob.dec.wsgify(RequestClass=webob.Request)
    def _dispatch(req):
        """Dispatch the request to the appropriate controller.

        Called by self._router after matching the incoming request to a route
        and putting the information into req.environ.  Either returns 404
        or the routed WSGI app's response.

        """
        print("step 2: RoutesMiddleware is invoked, calling our _dispatch back")
        
        match_dict = req.environ['wsgiorg.routing_args'][1]
        if not match_dict:
            return webob.exc.HTTPNotFound()
        app = match_dict['controller']
        return app
        
class RoutingTestCase(testtools.TestCase):

    def test_router(self):
        router = MyRouter()
        result = webob.Request.blank('/dummies').get_response(router)
        self.assertEqual(result.body, "getlist(), mykey=myvalue")
```

- 关于添加一个路由请求的操作参考这里[openstack-wsgi的route中增加api流程详解（os-networks）增加](http://blog.csdn.net/tantexian/article/details/37884037)




## 参考文章


[使用Python Routes + webob 的 REST API的测试程序.](http://www.itnose.net/detail/6098529.html)

[openstack-wsgi的route中增加api流程详解（os-networks）增加](http://blog.csdn.net/tantexian/article/details/37884037)

[浅析openstack中WSGI+Restful router“框架？”](http://www.maxwellxxx.com/openstackWsgi)


***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***
