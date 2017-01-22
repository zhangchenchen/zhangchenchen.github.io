---
layout: post
title:  "openstack系列-- benchmarking工具rally"
date:   2016-10-31 17:06:25
tags: openstack
---

## 目录结构：

[ralley是什么 ](#A)

[rally 安装以及快速引导 ](#B)

[rally 任务启动配置文件](#C)

[rally 插件](#D)




<a name="A"></a>

## ralley是什么

一句话概括，rally是一个测试openstack 性能的工具。它的存在主要回答这样一个问题：How does OpenStack work at scale? 意译一下就是：openstack 在负载比较大的规模下运转情况怎么样？
rally以一种插件式的形式工作，对openstack是无侵入的。通常会作为CI的一种功能集成到CI中去。

tips:
随着rally项目的不断发展，rally又衍生出了其他的一些功能，比如与tempest结合进行openstack的功能测试等。如下图

![rally action](http://7xrnwq.com1.z0.glb.clouddn.com/20161031Rally-Actions.png)

deploy(部署)功能并非是另一种部署方式，只是与devstack等部署方式以插件形式结合来简化工作而已。

<a name="B"></a>

## rally 安装以及快速引导

rally 安装比较简单，还可以跟Devstack 部署时一块安装，以及docker 安装[check this!](https://rally.readthedocs.io/en/latest/install.html)

安装完成之后我们就可以进行性能测试了，大致顺序如下：

 - 创建一个openstack 的rally测试环境：如果是有现成的openrc文件，那么直接source 一下，然后执行：  rally deployment create --fromenv --name=existing 就创建完成了。也可以将openrc文件中的内容写入一个json文件，假设取名为existing.json,执行如下命令： rally deployment create --file=existing.json --name=existing 。 执行rally deployment list 即可查看我们创建的deployment 以及哪一个正在被激活（正在使用）。
 ![rally deployment](http://7xrnwq.com1.z0.glb.clouddn.com/20161031rally2.png)
 通过 rally deployment check 可查看检测到的openstack服务：
 ![rally check](http://7xrnwq.com1.z0.glb.clouddn.com/20161031rally1.png)

 - 创建完一个deployment之后我们就可以进行测试了，我们需要指定一个json或yaml文件 来说明测试的内容， 在rally安装目录samples/tasks/scenarios 中有很多示例文件，比如samples/tasks/scenarios/nova 目录下就有nova对应的情景测试文件，例如boot-and-delete.json，就是启动虚拟机再删除操作，内容如下：

```json
{
    "NovaServers.boot_and_delete_server": [
        {
            "args": {
                "flavor": {
                    "name": "m1.tiny"
                },
                "image": {
                    "name": "^cirros.*uec$"
                },
                "force_delete": false
            },
            "runner": {
                "type": "constant",
                "times": 10,
                "concurrency": 2
            },
            "context": {
                "users": {
                    "tenants": 3,
                    "users_per_tenant": 2
                }
            }
        }
    ]
}
```

利用此文件启动任务： rally task start samples/tasks/scenarios/nova/boot-and-delete.json
还可以利用rally查看images ，flavors列表等。

 - rally提供了多种方式进行结果的查看，最直观的就是以html形式在浏览器中展现（需翻墙）：
  rally task report --out=report1.html --open 
  
  ![rally pic](http://7xrnwq.com1.z0.glb.clouddn.com/20161031rally3.png)


tips:执行任务时指定的json 文件格式如下：

```json
{
    "<ScenarioName1>": [<benchmark_config>, <benchmark_config2>, ...]
    "<ScenarioName2>": [<benchmark_config>, ...]
}
```

其中， <benchmark_config> 格式如下：

```json
{
    "args": { <scenario-specific arguments> },
    "runner": { <type of the runner and its specific parameters> },
    "context": { <contexts needed for this scenario> },
    "sla": 
    { <different SLA configs> }
}
```


<a name="C"></a>

## rally 任务启动配置文件

rally任务启动配置文件就是上文中提到过的json/yaml文件，其实可以通过此配置文件实现很多功能，比如：多任务配置，设置SLA(Service-Level Agreement,就是一个成功与否的基准，例如：max_seconds_per_iteration": 10)，利用jinja模板传参数进来，或利用jinja模板进行简单的逻辑控制等，[check this](http://rally.readthedocs.io/en/latest/tutorial/step_5_task_templates.html),不再赘述。 

<a name="D"></a>
## rally 插件

rally插件就是我们在json文件中配置的ScenarioName，通过这些插件的组合可以完成我们自己定义的要求。几个命令如下：

![rally plugin](http://7xrnwq.com1.z0.glb.clouddn.com/20161031rallyplugin.png)

关于rally与tempest的结合使用还不是很成熟，感兴趣的可以去试下，[check this!](http://rally.readthedocs.io/en/latest/tutorial/step_10_verifying_cloud_via_tempest.html)



## 参考文章

[openstack rally wiki](https://wiki.openstack.org/wiki/Rally)

[SDN使用 Rally 来实现 Openstack Tempest 测试](https://www.ibm.com/developerworks/cn/cloud/library/1604-rally-openstack-tempest/)


***END***
