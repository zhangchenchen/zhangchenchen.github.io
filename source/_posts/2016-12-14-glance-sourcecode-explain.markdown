---
layout: post
title:  "openstack系列--glance 源代码整体解读"
date:   2016-12-14 15:06:25
tags: openstack
---

## 目录结构：

[glance 介绍](#A)

[glance API 详解](#B)

[glance 整体代码解读 ](#C)



<a name="A"></a>

## glance 介绍

- openstack glance项目提供restful接口实现了虚拟机镜像的发现，注册，检索等功能。它本身不负责镜像的存储，提供driver接口，依赖于swift等其他项目完成镜像的存储。
- 因为版本不同，代码也可能不同，该版本为Openstack Liberty

 
<a name="B"></a>

## glance API 详解

- glance api 的版本目前如下：

 ![glance-api-version](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/20161214-glance-api-version.png)

  v1版本逐渐弃用，目前v2版本用的比较多，v3版本正在开发中。v1与v2版的区别还是比较大的，不只是添加了很多API。v1版本中glance-api与glance-registry都是一个WSGI server,glance-api负责接收用户的restful请求，如果该请求是与metedata相关的，则将其转发给glance-registry，glance-registry会解析请求并与数据库交互，如果该请求是与image自身存取相关的，则直接转发给store backend。V2版本中，glance-registry服务的内容被整合进了glance-api中，采用责任链的模式实现API的处理流程。在下面结合实例分析的时候会更清晰。
  以下示例图来自网上，正确性不保证：

 ![glance-api-version](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/20161214-glance-api-version.png)

 关于API 的详细介绍在这个页面[check this](http://developer.openstack.org/api-ref/image/v2/index.html)


<a name="C"></a>

## glance 整体代码解读


![glance code](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/20161214-glance-code.png)

- 看下setup.cfg中的启动脚本内容：

```
[entry_points]
console_scripts =
    glance-api = glance.cmd.api:main
    glance-cache-prefetcher = glance.cmd.cache_prefetcher:main
    glance-cache-pruner = glance.cmd.cache_pruner:main
    glance-cache-manage = glance.cmd.cache_manage:main
    glance-cache-cleaner = glance.cmd.cache_cleaner:main
    glance-control = glance.cmd.control:main
    glance-manage = glance.cmd.manage:main
    glance-registry = glance.cmd.registry:main
    glance-replicator = glance.cmd.replicator:main
    glance-scrubber = glance.cmd.scrubber:main
    glance-glare = glance.cmd.glare:main
    ..................
```

1，glance-cache-* :四个对image cache进行管理的工具，pruner用于执行一些周期性任务，cleaner用于清理cache文件释放空间。

2，glance-manage :执行对glance数据库的操作。

3，glance-replicator:实现镜像的复制功能

4，glance-scrubber:清理已经删除的image

5, glance-control:glance 提供了glance-api,glance-registry 两个WSGI Server,以及一个glance-scrubber后台服务进程，这里的glance-control 就是用于控制这三个服务进程（start,stop,restart）。


其实这里glance-api在初始时，还会导入一个叫做glance-store的项目，以便初始化后台的存储系统，glance通过glance-store这个框架所提供的接口，实现了对各种不同存储系统的支持。

- glance的主要操作对象是image，image的metedata存储在数据库，真实的image数据存储在后端存储系统（如swift,ceph）。

     1. image的主要metedata有：id,name,owner,size,location,disk_format,container_format(一般设置为bare),status。

     2. image的访问权限：

        - public 公共的：可以被所有的 tenant 使用。
        - private 私有的/项目的：只能被 image owner 所在的 tenant 使用。
        - shared 共享的：一个非共有的image 可以 共享给另外的 tenant，可通过member-* 操作来实现。
        - protected 受保护的：protected 的 image 不能被删除。
        
     3. image的status变更：

    ![image-status](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/574303-20160130210517989-1038953385.png)

- glance 中的task概念：在V2版本的API中提出了task的概念，关于为什么提出task的概念以可看这篇博客[Getting started with Tasks API in Glance](https://geetikabatra.wordpress.com/2015/06/15/getting-started-with-tasks-api-in-glance/)。
 
总结一下就是： 要给用户一个上传/下载镜像等操作的接口，但是目前的接口只是单纯地实现了功能，过程是不可控的，要是用户上传的不是镜像文件怎么办，甚至是恶意文件怎么办，执行过程中出现错误怎么办（没有反馈机制，用户是不可见的），本身镜像文件是非常大的，上传/下载占用太多时间怎么办。
 
因此针对这种耗时较长且不可控的操作，抽象出了task的概念。task是针对image的异步操作，本身具有一些属性如id,owner，status等。

task的status大致有：pending(task被创建，但未执行)，processing(task执行中),success(task成功结束),failure(执行失败)。

目前task支持的操作包括：import（用户上传镜像）,export（用户下载镜像），clone（region之间clone,注：紧紧跟随AWS啊）。具体的task api 可以看之前的参考页面。






## 参考文章

[openstack 设计与实现](https://book.douban.com/subject/26374647/)

[openstack glance github](https://github.com/openstack/glance)

[Image Service API ](http://developer.openstack.org/api-ref/image/v2/index.html#images)

[Getting started with Tasks API in Glance](https://geetikabatra.wordpress.com/2015/06/15/getting-started-with-tasks-api-in-glance/)

[wiki--glance-task-api](https://wiki.openstack.org/wiki/Glance-tasks-api)

***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***
