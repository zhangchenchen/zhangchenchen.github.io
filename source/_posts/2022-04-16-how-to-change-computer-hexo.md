---
layout: post
title:  "工具--更换电脑时如何迁移hexo"
date:   2021-04-16 16:48:30
tags:
  - 工具
---


## 前言

切换电脑后如何将Hexo迁移到新电脑无缝编辑使用。


##  新电脑安装hexo

- 安装node

- 安装git

- 安装hexo：
```shell
npm install -g hexo-cli
```

- clone远程仓库hexo分支到本地:
```shell
git clone git@github.com:username/username.github.io.git
```

- 项目目录下面安装依赖：
```shell
npm install
```

- 本地生成网站并开启博客服务器：
```shell
hexo g & hexo s
```
如果一切正常，此时打开浏览器输入http://localhost:4000/ 已经可以看到博客正常运行了


## 参考文章

[hexo博客同步管理及迁移](https://www.jianshu.com/p/fceaf373d797)


***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***
