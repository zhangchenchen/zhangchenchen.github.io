---
layout: post
title:  "other--博客从jekyll搬到hexo记录"
date:   2017-02-03 13:54:25
tags: other
---

## 目录结构：

[引出](#A)

[blog 搬家的步骤](#B)




<a name="A"></a>

## 引出

- 之前写blog的初衷是作为自己以后翻阅的记录，所以就用了一个非常简单的jekyll theme，随着写的越来越多，因为这个theme没有归档功能，发现越来越不好打理，所以决定换个主题。
- 网上搜了一下后，发现hexo可以实现本地预览的功能，就决定用hexo来搭，[next](https://github.com/iissnan/hexo-theme-next)这个主题是个比较经典的主题，作者的维护也一直很棒，甚至出了一个[用户手册](http://theme-next.iissnan.com/),基本能实现我的所有需求。
- 因为原blog就是搭在github上，所以现在所要做的工作就是利用hexo做的blog替换原有的在github page上的blog。



<a name="B"></a>

## blog 搬家的步骤


1. 删除原github page 项目。注意保留本地之前写的blog。
2. 安装hexo，并按照[用户手册](https://hexo.io/zh-cn/docs/configuration.html)做一些初始化的工作。
3. 做blog的迁移工作，将blog的md文件复制到 source/_post文件夹，并在 _config.yml 中修改 new_post_name 参数。
```bash
new_post_name: :year-:month-:day-:title.md
```
4. 修改blog中的部分md格式，比如代码高亮格式等。
5. 下载 next theme 到themes文件夹，并配置该theme，参考[next theme](http://theme-next.iissnan.com/getting-started.html)。注意如果是使用git clone下来的，那么会存在两个远程分支（github page 一个，next theme一个），容易出错，所以最好还是直接下载下来放到themes文件夹（当然，也可以使用git submodules来管理）。
6. git bash 中运行依次 hexo g （生成静态文件），hexo s（启动本地服务器） ，然后本地查看。
7. 接下来需要将blog部署到github pages。因为我需要在家里和公司的电脑进行博客的更新，所以涉及到一个协作的问题。github page 上面挂的都是一些静态的文件，而如果修改主题或者配置的话，就没法做到同步了。搜了一下，知乎上找到一个比较完美的解决方案，通过创建两个分支来解决，参考[使用hexo，如果换了电脑怎么更新博客？](https://www.zhihu.com/question/21193762)。





## 参考文章

[Next theme](http://theme-next.iissnan.com/getting-started.html)


[hexo doc](https://hexo.io/zh-cn/docs/)


[使用hexo，如果换了电脑怎么更新博客？](https://www.zhihu.com/question/21193762)

***END***
