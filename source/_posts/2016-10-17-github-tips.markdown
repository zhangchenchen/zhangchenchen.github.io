---
layout: post
title:  "git系列--关于github 的碎碎念"
date:   2016-10-17 22:01:21
tags: git
---


## 目录结构

[github 快捷键 ](#A)

[github trending && github subscribe ](#B)

[为开源项目贡献代码](#C)

[git 常用命令](#D)


<a name="A"></a>

## github 快捷键

逛v2ex的时候逛到了这个帖子，[录了几个有关 Github 的视频，我觉得你也应该知道这些](https://www.v2ex.com/t/313267#reply54),很有意思,把视频看了一遍，都不是很长，强烈推荐。顺手记录下这些信息。
首先是github的一些快捷键：

 - 项目中搜索含有某个关键字的文件：在github 项目的code目录按下 "t",即可进入搜索页面，如下图：
 
 ![search](http://7xrnwq.com1.z0.glb.clouddn.com/20161017github-search.jpg)

 - 列出github的快捷键：在项目目录按下 “？” 即可出现：

![github shortcut](http://7xrnwq.com1.z0.glb.clouddn.com/20161017shortcut.jpg)

 - 进入 issue页面：按下“g” 与 “i”键，就是goto issue 的缩写
 - 进入code页面：按下“g”与“c”键，goto code

其他的按下“？”自己探索吧。


<a name="B"></a>

## github trending && github subscribe

 - github trending:进入这个页面[github trending](github.com/trending)就可以看到最近按照stars数排序靠前的比较优秀的项目，可以按照语言分类。
 - github subscribe:进入这个页面[github subscribe](https://github.com/explore/subscribe),我们可以订阅我们关注的大牛，他们的动态会按周/月发送到你的邮箱。

<a name="C"></a>

## 为开源项目贡献代码

关于这个，强烈推荐看下视频吧，直观易懂。这里简短记录下过程：

 - fork and clone 
 - github remote -add upstream ssh@xxxxxxxx.git :标示我们fork的上级，master分支要与upstream分支保持一致
 - github checkout -b feature/bug-fixed : 创建并切换分支，在此分支上编码。
 - git add && commit 
 - git checkout master && git pull upstream master：在push之前我们先要保证master与upstream一致。
 - git checkout feature/bug-fixed && git rebase master：将在bug-fixed上的commit log 打到master分支的最上面。
 - git push origin featue/bug-fixed
 - git pull request: 现在可以向开源作者pull request了

<a name="D"></a>

## git 常用命令

- 切换分支后，清理本地目录：git reset --hard HEAD && git clean -i 

- 不详细写了，以后用得着直接去这个页面找吧：
[git-tips](https://github.com/521xueweihan/git-tips)

## 参考文章

[v2ex](https://www.v2ex.com/t/313267#reply54)

[git-tips](https://github.com/521xueweihan/git-tips)


***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***

