---
layout: post
title:  "openstack-- 如何调试openstack rest API"
date:   2017-02-22 15:18:25
tags: openstack
---



## 引出

在学习openstack 的过程中，基本都是这样一个过程，先是通过 dashboard 对openstack 有一个感性的认识，之后通过命令行了解openstack 更多的功能，但openstack 的内部实现还是一个黑盒。这时就需要了解 openstack 提供的最原始的 rest API, rest API 不仅是提供给终端 user使用，它内部组件的通信机制也用的rest API ，当然还有AMQP,关于 rest与AMQP 的比较见[这篇文章](https://zhangchenchen.github.io/2017/01/10/something-about-http&tcp&amqp/) 。本篇文章主要讲如何使用 postman 调试openstack 的rest api，之前会花一些篇幅讲一点openstack api version 的知识。


## openstack api 版本演变

openstack 自开源以来发展还是挺迅猛的，对应各组件的api version演变也很快，特别是 nova, cinder, keystone等组件的api。在最新的官方文档中，也就是 N 版本的openstack,目前使用的API(只列出一些常用的) 为 ：
- Bare Metal API v1 (microversions)
- Block Storage API v3 (microversions)
- Clustering API v1
- Compute API (microversions)
- Container Infrastructure Management API (microversions)
- Identity API v3
- Identity API v3 extensions
- Image service API v2
- Messaging API v2
- Networking API v2.0
- Object Storage API v1

还支持或者兼容的older api为：
- Block Storage API v2

废弃的api 为：
- Block Storage API v1
- Identity API v2.0
- Identity admin API v2.0
- Identity API v2.0 extensions
- Image service API v1

### microversion 解释


这里简单解释下 microversion,microversion的使用是为了向后兼容性，因为openstack的版本迭代比较快，当我们对openstack进行版本升级后，对应的client 或其他调用openstack api的系统因为某些原因没法升级，仍然依赖之前的API，这个时候就可以使用microversion来解决这个问题。
举个栗子，比如Compute API，我们先通过一个 get请求( youropenstackIP:8774/ )获取支持的version 版本，返回如下：

```json
{
    "versions": [
        {
            "id": "v2.0",
            "links": [
                {
                    "href": "http://openstack.example.com/v2/",
                    "rel": "self"
                }
            ],
            "status": "SUPPORTED",
            "version": "",
            "min_version": "",
            "updated": "2011-01-21T11:33:21Z"
        },
        {
            "id": "v2.1",
            "links": [
                {
                    "href": "http://openstack.example.com/v2.1/",
                    "rel": "self"
                }
            ],
            "status": "CURRENT",
            "version": "2.42",
            "min_version": "2.1",
            "updated": "2013-07-23T11:33:21Z"
        }
    ]
}
```
可以看到支持 V2 版本以及V2.1 的microversion,该 microversion，支持从 V2.1 到 V2.42的所有版本的API，但如果我们使用的话还是需要指定一个版本，在 header中加一个参数：
```json
X-OpenStack-Nova-API-Version: 2.4
```


### keystone v3 简介

这里简单说下keystone v3，因为 keystone v3相对于 v2版本来说增加了很多新的特性，这里简单记录一下。

v2版本对用户的权限管理以每一个用户为单位，需要对每一个用户进行角色分配，并不存在一种对一组用户进行统一管理的方案，v3版本中引入了 Domain 和 Group的概念，将 tenant改为project,从而实现对一组用户进行统一管理。

直接引用[OpenStack Keystone V3 简介](http://www.ibm.com/developerworks/cn/cloud/library/1506_yuwz_keystonev3/index.html)

 >V3 利用 Domain 实现真正的多租户（multi-tenancy）架构，Domain 担任 Project 的高层容器。云服务的客户是 Domain 的所有者，他们可以在自己的 Domain 中创建多个 Projects、Users、Groups 和 Roles。通过引入 Domain，云服务客户可以对其拥有的多个 Project 进行统一管理，而不必再向过去那样对每一个 Project 进行单独管理。
 Group 是一组 Users 的容器，可以向 Group 中添加用户，并直接给 Group 分配角色，那么在这个 Group 中的所有用户就都拥有了 Group 所拥有的角色权限。通过引入 Group 的概念，Keystone V3 实现了对用户组的管理，达到了同时管理一组用户权限的目的。这与 V2 中直接向 User/Project 指定 Role 不同，使得对云服务进行管理更加便捷。



## 使用 postman 调试openstack 的rest api

### openstack 常用组件默认监听端口

在使用postman调试之前，先列出openstack 中常用组件默认监听的端口：

![default-port](http://oeptotikb.bkt.clouddn.com/2017-02-22-openstack-default-port.png)

其他服务组件的默认监听端口：

![other-default-port](http://oeptotikb.bkt.clouddn.com/2017-02-22-openstack-extend-default-port.png)


### 使用 postman 调试 openstack rest api

调试 rest api 的客户端工具有很多，官网用的是curl，但不是很直观，postman 算是颜值比较高的一款 http api调试工具。我比较习惯于使用 chrome插件版本的postman，安装完毕后，开始进行调试：

- 向 keystone 发送post请求获取 token 以及 service endpoint:

    resuest headers 内容如下：
![headers](http://oeptotikb.bkt.clouddn.com/2.png)

    resuest body 内容如下：

![body](http://oeptotikb.bkt.clouddn.com/2017-02-22-header-body.png) 
    返回结果如下：

 ```json
{
  "access": {
    "token": {
      "issued_at": "2017-02-22T08:17:13.025922",
      "expires": "2017-02-23T08:17:12Z",
      "id": "1213f82cd41f472ba4291e75a2de8288",
      "tenant": {
        "description": "Admin Tenant",
        "enabled": true,
        "id": "585324ae7e934e2eb50d84ad7aee3179",
        "name": "admin"
      },
      "audit_ids": [
        "T7g-8ZGYTZOUN5oEI-2q5w"
      ]
    },
    "serviceCatalog": [
      {
        "endpoints": [
          {
            "adminURL": "http://192.168.24.161:8774/v2/585324ae7e934e2eb50d84ad7aee3179",
            "region": "RegionOne",
            "internalURL": "http://192.168.24.161:8774/v2/585324ae7e934e2eb50d84ad7aee3179",
            "id": "899cc922536f4038922bac198616d086",
            "publicURL": "http://192.168.24.161:8774/v2/585324ae7e934e2eb50d84ad7aee3179"
          },
          {
            "adminURL": "http://192.168.24.100:8774/v2/585324ae7e934e2eb50d84ad7aee3179",
            "region": "testVlan",
            "internalURL": "http://192.168.24.100:8774/v2/585324ae7e934e2eb50d84ad7aee3179",
            "id": "2e4cf2fac2be47a4936ba19e15894030",
            "publicURL": "http://192.168.24.100:8774/v2/585324ae7e934e2eb50d84ad7aee3179"
          }
        ],
        "endpoints_links": [],
        "type": "compute",
        "name": "nova"
      },
      ........................................
    ],
      ]
    }
  }
}
 ```
   
   可以看到 token id 即我们要的token标示，而且下一步需要的url即endpoint也已经出现在该json串中，以 compute service 的endpoint 为例，获取 虚拟机列表。

- 向 nova 发送 获取虚拟机列表的rest 请求，将上一步获取到的 token id填入 X-Auth-Token中。

 ![nova-rest](http://oeptotikb.bkt.clouddn.com/2017-02-22nova-auth.png)

   发送该请求即可得到我们的结果：

```json
{
  "servers": [
    {
      "id": "712ea870-9ac7-467d-a2d7-5e3caa8499a6",
      "links": [
        {
          "href": "http://192.168.24.161:8774/v2/ca36ea9826b04643897f19b7c03be011/servers/712ea870-9ac7-467d-a2d7-5e3caa8499a6",
          "rel": "self"
        },
        {
          "href": "http://192.168.24.161:8774/ca36ea9826b04643897f19b7c03be011/servers/712ea870-9ac7-467d-a2d7-5e3caa8499a6",
          "rel": "bookmark"
        }
      ],
      "name": "test"
    },
    {
      "id": "257a2852-940e-4271-b188-fcd4d0a8c10c",
      "links": [
        {
          "href": "http://192.168.24.161:8774/v2/ca36ea9826b04643897f19b7c03be011/servers/257a2852-940e-4271-b188-fcd4d0a8c10c",
          "rel": "self"
        },
        {
          "href": "http://192.168.24.161:8774/ca36ea9826b04643897f19b7c03be011/servers/257a2852-940e-4271-b188-fcd4d0a8c10c",
          "rel": "bookmark"
        }
      ],
      "name": "testmime9"
    }
  ]
}
```

  其他请求类似。



## 参考文章

[OpenStack API versions](https://developer.openstack.org/api-guide/quick-start/)

[penStack API Documentation](https://developer.openstack.org/api-guide/quick-start/api-quick-start.html)


[OpenStack DocumentationAppendix Firewalls and default ports](https://docs.openstack.org/kilo/config-reference/content/firewalls-default-ports.html)

***END***
