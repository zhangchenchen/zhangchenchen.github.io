---
layout: post
title:  "Docker-- 关于Harbor 的一些记录"
date:   2017-08-03 11:59:11
tags: 
  - Docker
  - Harbor
---


##  Harbor 简介

[Harbor](https://github.com/vmware/harbor) 是 Vmware China研发开源的企业级私有容器Registry,基于docker官方的解决方案[Distribution](https://github.com/docker/distribution)。目前来说，已经成为企业级私有容器仓库的首选（主要是可供选择的本来就不多，可能是因为镜像管理不像容器编排那样是必争之地）。相对于Docker Distribution，Harbor添加了安全，认证，管理等功能，性能以及安全性都得到提升，主要 features如下：

- 基于角色的访问控制：用户和镜像仓库通过project来组织管理，一个用户对同一project的不同镜像仓库会有不同处理权限。
- 基于策略的镜像复制：多个registry实例可以实现镜像同步复制，对于load-balancing,高可用，多数据中心，混合云的情景非常有用。
- 支持 LDAP/AD
- 镜像删除 & 垃圾回收
- 镜像的认证
- 友好的UI
- 日志审计：所有的操作都是可追踪的
- RESTful API
- 易部署 

## Harbor 架构与原理

### Harbor 整体架构

Harbor 是以容器的方式运行，以docker-compose的规范形式组织各个组件，并通过docker-compose工具进行启停。
Harbor共有五个组件，分别如下：
- Proxy:Harbor服务的所有请求都由该服务接受并转发，其实就是一个前置的反向代理Nginx，类似于微服务概念中的API-gateway.
- Registry: 即docker 官方的Registry镜像生成的容器示例，真正负责存贮镜像的地方，处理docker pull/push。针对不同的用户对不同的镜像操作权限不同，registry强制每个请求必须含有一个token以验证权限，如果没有token，会返回一个token服务地址。
- Core Services: 主要提供UI,webhook(设置在registry上以获取镜像状态)，token服务。
- Database:数据库服务Mysql，存储用户，权限，审计日志，镜像信息等。
- Log Collector: 日志收集，跑一个Rsylogd服务。

架构图如下：

![harbor-arch](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/2017-08-03.harbor-arc.jpg)

这五个容器之间通过docker-link相连，即通过容器名字互相访问，暴露Proxy服务的端口给终端用户访问。

### Harbor 工作原理

以docker login 与 docker push为例讲解：

客户端 输入docker login 之后，流程如下图：
![docker-login-flow](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/2017-08-03-harbor-flow1.jpg)

(a) 首先，这个登录请求会被Proxy容器接收到，根据预先设置的匹配规则，该请求会被转发给后端Registry容器。
(b) Registry接收到请求后，解析请求，因为配置了基于token的认证，所以会查找token，发现请求没有token 后，返回错误代码401以及token服务的地址URL。
(c) Docker客户端接收到错误请求后，转而向token服务地址发送请求，并根据HTTP协议的BasicAuthentication 规范，将用户名密码组合并编码，放在请求头部(header)。
(d) 同样，该请求会先发到Proxy容器，继而转发给ui/token的容器,该容器接受请求，将请求头解码，获取到用户名密码。
(e) ui/token的容器获取到用户名密码后，通过查询数据库进行比对验证(如果是LDAP 的认证方式,就是与LDAP服务进行校验)，比对成功后，返回成功的状态码，并用密钥生成token，一并发送给Docker客户端。

客户端 登陆成功后，输入docker push xxxxxx 之后，流程如下图（便于说明省略Docker client与Proxy之间通信）：

![docker-push-flow](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/2017-08-03-harbor-flow-2.jpg)

(a) 同样，首先与Registery通信，返回一个token服务的地址URL.
(b) Docker客户端会与token服务通信，指明要申请一个push image操作的token。
(c) token服务访问数据库验证当前用户是否有该操作的权限，如果有，会将image信息以及push操作进行编码，用私钥签名，生成token返回给Docker客户端。
(d) Docker客户端再次与Registry通信，不过这次会将token放到请求header中，Registry收到请求后利用公钥解码并核对，核对成功，便可以开始push 操作了。


## Harbor 安装

比较简单：

1. 准备：python>=2.7, docker>=1.10, docker-compose>=1.6.0
2. 下载离线安装包
3. 修改配置文件 harbor.cfg
4. 执行脚本install.sh 

参考[Installation and Configuration Guide](https://github.com/vmware/harbor/blob/master/docs/installation_guide.md)


## Harbor 高可用

对于Harbor高可用方案，目前并没有最佳实践，不过我看issues上有不少相关内容，可以参考[Harbor HA feature design proposal/discussion](https://github.com/vmware/harbor/issues/327)。
其实，私有云相对来说对镜像的请求并非高频，在做HA的时候还是结合实际情况，切勿为了HA而HA,还要综合考量成本，安全等因素。

这部分内容暂且留个坑，以后再写。



## 参考文章


[VMware Harbor：基于 Docker Distribution 的企业级 Registry 服务](https://segmentfault.com/a/1190000007705296)

[基于Harbor和CephFS搭建高可用Private Registr](http://tonybai.com/2017/06/09/setup-a-high-availability-private-registry-based-on-harbor-and-cephfs/)

[vmware/harbor](https://github.com/vmware/harbor)

[Docker 企业级私有镜像仓库 Harbor 部署](http://jaminzhang.github.io/docker/Enterprise-class-private-Docker-Container-Registry-Harbor-deploying/)

***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***