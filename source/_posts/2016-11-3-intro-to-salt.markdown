---
layout: post
title:  "devops系列-- 运维利器SaltStalk介绍"
date:   2016-11-3 13:06:25
tags: 
  - saltstack
  - devops
---


<a name="A"></a>

## SaltStalk是什么

salt官网给出的介绍简洁明了：

- 一个配置管理系统，能够维护预定义状态的远程节点(比如，确保指定的报被安装，指定的服务在运行)
- 一个分布式远程执行系统，用来在远程节点（可以是单个节点，也可以是任意规则挑选出来的节点）上执行命令和查询数据。开发其的目的是为远程执行提供最好的解决方案，并使远程执行变得更好，更快，更简单。

其实，除了saltstack,还有其他的同类型软件供我们选择，比如puppet、chef、ansible、fabric等,关于其中的区别以及优劣势可以参考这里，[Ansible vs. Chef vs. Fabric vs. Puppet vs. SaltStack](http://blog.takipi.com/deployment-management-tools-chef-vs-puppet-vs-ansible-vs-saltstack-vs-fabric/)
稍微总结下：chef,puppet 是出现时间比较早的系统，对于比较重视成熟性和稳定性的公司来说比较合适。ansible与saltstack比较适合快速灵活的生产部署环境，且适合没有太多额外要求，生产环境操作系统一致的情况下。fabric比较适合规模比较小的环境，属于入门级别的配置管理系统。

这里有一个视频更细粒度的（可用性，互操作性，扩展性等）比较了一下，[Chef vs Puppet vs Ansible vs SaltStack | Configuration Management Tools Comparison ](https://www.youtube.com/watch?v=OmRxKQHtDbY&t=1060s)



<a name="B"></a>

## SaltStalk 安装

安装比较简单，[check this](http://docs.saltstack.cn/topics/installation/index.html),不再赘述。


<a name="C"></a>

## SaltStalk 配置

saltstalk是典型的CS架构，主要分两种角色，master和minions,master负责minions的配置管理以及远程执行。一般来说，master的配置文件位于/etc/salt/master，minion的配置文件在相应机器的/etc/salt/minion,只需在minion配置文件中指定master指向即可运行。

[master的配置项](http://docs.saltstack.cn/ref/configuration/master.html)非常多，大致包括以下几项：
 - 主要配置：包括网络接口，提供服务的端口，master id,最大打开文件数，工作线程个数，返回端口，缓存目录，各种缓存配置（minions数据缓存，作业缓存等）
 - master 安全相关配置：主要是针对PKI验证的一些配置。
 - master模块管理相关配置
 - master state system 相关配置
 - pillar 相关配置
 - 日志相关配置
 - windows软件源相关配置

[minion的配置项](http://docs.saltstack.cn/ref/configuration/minion.html)跟master大致类似，不再赘述。


其他：
 - 新版saltstack 中minion有一个minion_blackout的配置，该选项设为true后，该minion不会执行除saltutil.refresh_pillar之外的其他所有命令。
 - saltstack有一个访问控制系统对于非admin用户进行细粒度的访问控制。[check this](http://docs.saltstack.cn/topics/eauth/access_control.html)
 - job管理器：可以实现对minion job的发送信号，定时调度等
 - job 返回数据管理：系统默认返回数据存储在Job cache（默认是在/var/cache/salt/master/jobs目录）中，还有其他两种存储模式：一种是External Job Cache，如下图

![External Job Cache](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/20161107external-job-cache.png)

另一种是Master Job Cache，如下图：

![Master Job Cache](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/20161107master-job-cache.png)

 - returner: 我们可以在master端执行远程指令时指定returner,这样minion返回的数据不仅会返回到master，也会返回到我们指定的returner。而且我们也可以按照要求实现一个returner，来替换我们的Default Job Cache。




<a name="D"></a>

## SaltStalk 一些概念解释

### sls 文件

salt state 文件的简称，saltstalk实现配置管理的核心部分，描述了系统的目标状态，一般遵循yaml格式，参考[YAML 语言教程](http://www.ruanyifeng.com/blog/2016/07/yaml.html)。

首先是配置管理的入口文件top.sls，默认在/srv/salt/目录下。

```yaml
base:          # 默认环境变量
  '*':         # 通过正则进行匹配minion
    - apache    # 需要自己写的state.sls模块名 
  
  my_group:             #通过分组名去进行匹配 须定义match:nodegroup
    - match: nodegroup  
    - nginx

  'os:CentOs':        #通过grains模块去匹配，须定义match:grain
    - match: grain
    - nginx
```

再看下需要自己写的webserver.sls文件，默认位于/srv/salt/apache.sls。当然，还有其他组织形式，常用的还有/srv/salt/apache/init.sls 形式。

```yaml
apache:           # 标签定义
  pkg:               # 状态定义，这里使用（pkg state module）
    - installed      # 安装nginx（yum安装）
  service.running:   # 保持服务是启动状态
    - enable: True
    - reload: True
    - require:
      - file: /etc/init.d/httpd
    - watch:                 #检测下面配置文件，有变动，立马执行上述/etc/init.d/httpd 命令reload操作
      - file: /etc/apache/httpd.conf
      - pkg: nginx
```


state 之间的逻辑关系一般有三种：

- require :依赖某个state，在运行此state前，先运行依赖的state，依赖可以有多个
- watch :在某个state变化时运行此模块，watch除具备require功能外，还增了关注状态的功能.
- order：优先级比require和watch低，有order指定的state比没有order指定的优先级高

一般对某个minion执行具体state的时候，我们可以执行 
```bash
$ salt minion-1 state.sls apache 
```

当然，如果是执行所有的sts文件，则是如下命令：

```bash
$ salt minion-1 state.highstate   # test=True 参数可以测试安装，并不真正安装
```


### grain vs pillar vs mine


如果说sls文件是配置管理的骨架或框架的话，那么grain与pillar就是填充骨架的血与肉。其实他们就是saltstack自己定义的一些数据，通过这些数据定制我们的系统配置。

- grains 存储在minion一端，包括minion自己生成的一些信息，比如操作系统，内存，磁盘信息，cpu架构等等。当然我们也可以自己定制minion 的grains信息。通常做法是在默认/etc/salt/grains中定义。
- pillar 是存储 在master端，完全是用户自定义的一些动态数据。一般存储在master端/srv/pillar 目录下。组织形式类似salt state。 
- 两者区别：一般来说，变化较少或不变的数据存储在grains中，一些敏感数据（如各种密码），易变数据，以及涉及到minion 配置的数据一般存储在pillar中。
- mine 更像是以上两种形式的结合，它是定期从minions 收集的数据，传给master并存储在master端，这样，所有的minions都能获取到。mine数据是有时效性的，每隔一段时间便会更新。mine可以存储在mine配置文件中，不过更多的是将它放在pillar中，只需定义mine_functions 关键字即可。官网示例参考[EXAMPLE](https://docs.saltstack.com/en/latest/topics/mine/index.html#example)


两者的常用操作如下：

```bash
$ salt '*' grains.items   # 查看所有 grains键值对

$ salt '*' grains.get os  # 查看 grains中的 os对应的值

$ salt '*' saltutil.sync_grains # 同步所有 grains数据

$ salt '*' pillar.items   # 查看所有 pillar键值对

$ salt '*' pillar.get data  # 查看 pillar中的 data对应的值

$ salt '*' saltutil.refresh_pillar # 刷新所有 pilllar数据

$ salt '*' state.apply my_sls_file pillar='{"hello": "world"}' # 命令行更改或增加pillar
```




### execution modules 

也就是运程执行时需要 minions执行的函数，通常salt已经封装好了大量的modules,可以通过 salt '*' sys.doc 查看所有modules的doc。当然也可以自己写modules，下文会给出示例。


### runner 

runner 本质是可定制化的Python 脚本，与execution modules 类似，不过是运行在master节点，可以使用salt-run 命令使用，非常方便（最新版的salt-run也可以运行modules 了，命令：salt-run salt.cmd test.ping ,）。
直接举个栗子：
首先指定runner脚本的位置，在master配置文件中指定：runner_dirs: [/srv/runner/]
然后就可以在指定目录下写Python 脚本了。

minions.py ：
```python
# Import salt modules
import salt.client

def up():
    '''
    Print a list of all of the minions that are up
    '''
    client = salt.client.LocalClient(__opts__['conf_file'])
    minions = client.cmd('*', 'test.ping', timeout=1)
    for minion in sorted(minions):
        print minion

```

使用命令行： salt-run minions.up  即可调用。
注：有两种模式，默认为同步模式，即有数据返回后才显示，还有一种异步模式，即立即返回不显示数据，可以指定returner将数据返回到指定容器中。


### salt engines

salt engines 是一个利用salt 并独立长时间运行的进程。有以下特点：

- 可以获取到salt的各种配置项，modules以及runner
- 以独立进程运行，由salt监视，一旦挂掉后由salt负责重启。
- 可以运行在master和minion上。

<a name="E"></a>

## SaltStalk 使用

### 远程执行

saltstack 最基本的功能之一，在master节点实现对minions节点的远程控制与执行，命令格式如下：

```bash
$ salt '<target>' <function> [arguments]
```

- target 指定 minions,可以有多种方式匹配minions，正则匹配，group匹配，grains匹配，pillar匹配等。
- function 就是前面讲的execution modules ，后面可以加参数。

下面介绍下execution modules 的定制开发。
首先确定一下modules的位置，默认位于/srv/salt/_modules目录下，可以通过以下命令实现modules的同步。
```bash
$ salt '*' saltutil.sync_modules

$ salt '*' saltutil.sync_all

$ salt '*' state.apply
```
通常modules的脚本会使用zip压缩文件，[官网示例](https://docs.saltstack.com/en/latest/ref/modules/index.html#creating-a-zip-archive-module)

脚本中可以调用salt本身的module以及调用grains数据。同时注意str 类型 与unicode的转换。



### 配置管理

配置管理功能是在远程执行的基础上建立起来的，Salt 状态系统的核心是就是上面提到过的SLS文件。SLS表示系统将会是什么样的一种状态（比如安装什么服务，服务是否启动，服务配置等等），而且是以一种很简单的格式来包含这些数据。
下面是一个写SLS文件的示例：

- 首先是在master配置文件中定义三个环境：

```bash
file_roots:
  base:
    - /srv/salt/prod
  qa:
    - /srv/salt/qa
    - /srv/salt/prod
  dev:
    - /srv/salt/dev
    - /srv/salt/qa
    - /srv/salt/prod
```

简单解释下：这里定义了三个环境，base环境就是生产环境，它只有一个salt根目录，qa环境可以获取到两个salt根目录下的配置文件，按照上下顺序优先采用 /srv/salt/qa 目录下的配置文件，dev环境同理。通常我们开发时，会首先在开发环境部署，新的SLS文件会存储在 /srv/salt/dev 环境中，然后将改变push到对应的开发机器。开发完成后，将新的SLS文件 复制到/srv/salt/qa 目录，push到对应的测试机器以供测试。最后测试完成后，才会把SLS文件复制到/srv/salt/prod 目录，push到生产环境的机器。

- 接下来开始编写SLS文件了，首先编写各环境根目录下的top文件,因为 /srv/salt/prod目录下的文件是所有环境都能获取到的,所以只需在该目录下编写top文件即可。

```yaml
base:
  'roles:prod':
    - match: grain
    - apache
qa:
  'roles:qa':
    - match: grain
    - apache
dev:
  'roles:dev':
    - match: grain
    - apache
```

这里我们是通过在 grain 中添加roles 键值对，然后通过grain进行匹配，当然也可以通过其他方法匹配（pillar,group，正则等）。apache是之后我们需要编写SLS文件，不过这在这之前，先在各minions 添加grain roles键值对。

- 在对应minions 中添加grain roles键值对，默认编辑/etc/salt/grains ,添加 roles: prod 即可。添加完后，master 执行 salt '*' saltutil.sync_grains 同步命令。
- 接下来开始编写apache的sls 文件。

/srv/salt/prod/apache/init.sls
```yaml
apache:
  pkg.installed:
    - name: httpd
  service.running:
    - name: httpd
    - require:
      - pkg: apache

/var/www/html/index.html:
  file.managed:
    - source: salt://apache/index.html
    - require:
      - pkg: apache
```

index.html 是我们的一个测试网页，放在/srv/salt/prod/apache/目录下。

- 接下来在 master端执行 salt '*' state.apply 命令 即可完成对应minions 端apache的安装，启动等工作。浏览器输入minion ip,可以查看到测试网页。

- 如果该过程中出现错误，多半是SLS文件写错了，可以查看 minion日志，如果日志不详细，可以先关掉salt-minion，然后运行 salt-minion -l debug ，再复现一次错误，便可查看到更详细日志。


### event/reactor

event/reactor 是saltstack 中的两个系统。两者结合我们可以定制一些自动化的功能，比如，minion端的salt-minion服务重启后，master端立马同步grains，pillars信息。
event系统是一个本地的ZeroMQ PUB接口, 用于产生salt events.这个event总线是一个开放的系统,用于发送给Salt和其他系统发送关于操作的通告信息.event系统产生event有一个严格的标准. 每一个event有一个 tag . event tags用于快速过滤events. 每一个event有一个数据结构附加在tag后. 这个数据结构是个字典, 包含关于本event的信息。
reactor 系统会结合sls文件在master端去匹配event tags，匹配成功后执行相应的sls文件。

暂时还没有碰到此类需求，回头再看吧。

<a name="F"></a>

## SaltStalk 架构

关于架构部分，内容比较繁杂，通常会涉及到multimaster,multimaster with failover,salt-syndic等，后续有需求了再研究吧。

参考[SALTSTACK ARCHITECTURE](http://docs.saltstack.cn/topics/topology/index.html)





## 参考文章

[saltstalk doc](https://docs.saltstack.com/en/latest/topics/using_salt.html)

[Saltstack SLS文件解读](https://my.oschina.net/u/877567/blog/183959)

[Saltstack自动化（五）sls文件使用](http://www.361way.com/salt-states/5350.html)


***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***
