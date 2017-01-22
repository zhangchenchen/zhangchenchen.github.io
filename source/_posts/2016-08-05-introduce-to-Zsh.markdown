---
layout: post
title:  "shell脚本系列--Zsh简介与使用"
date:   2016-08-05 10:30:25
tags: shell
---

Zsh同bash一样，是一款功能强大的终端（shell）软件，只不过bash是大部分Linux发行版默认的shell，而Zsh需要手动安装（mac默认已安装）。关于Zsh的更多介绍请看 [wiki](https://wiki.archlinux.org/index.php/Zsh_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))。

Zsh相比于bash有很多优势，两者的对比请看 [why zsh is cooler than your shell](http://www.slideshare.net/brendon_jag/why-zsh-is-cooler-than-your-shell?next_slideshow=1)
概括一下就是：高效，自动补全更优秀，可定制型高。借用官网上一句话：if you want you hand dirty, this is definitely your choice!

而且随着开源项目 [oh my zsh](https://github.com/robbyrussell/oh-my-zsh)的火爆，zsh的各种主题以及插件也很齐全，适应一段时间后用起来得心应手。

当然，如果生产环境是分布式的情况下，还是建议规矩的用bash，毕竟来回切换的成本也是不小的。

 ***

接下来开始简单说下zsh的安装以及试用。

 1. 准备工作
- Unix-based 机器
- 已安装 curl或者wget ,git
 
 2. 安装Zsh
  - 两种安装方法：借用包管理工具（如：apt-get install zsh），源码安装（略）
  - 安装完成后输入 *zsh --version* 出现相应的版本号即安装成功
  - 输入 *chsh -s $(which zsh)* 更改默认shell 为zsh
  - 退出之后再次登录即进入Zsh

 3. 安装Oh My Zsh（两种方法）
  - 第一种，通过curl:

 ```bash
   sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
 ```


  - 第二种，通过wget:  

 ```bash
  sh -c "$(wget https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh -O -)"
 ```

 4. Oh My Zsh使用简介
  - 插件：zsh的配置文件为 *~/.zshrc* 相应的插件配置选项为  *plugins=(git bundler osx rake ruby)*  
  - 主题：在配置文件中的配置选项为： *ZSH_THEME="robbyrussell"*  
  - 插件与主题都是可以配置的，可供配置的插件以及列表在这[Extern themes](https://github.com/robbyrussell/oh-my-zsh/wiki/External-themes) , [External-plugins](https://github.com/robbyrussell/oh-my-zsh/wiki/External-plugins)


## 参考文章

[Zsh wiki](https://wiki.archlinux.org/index.php/Zsh_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))

[oh my zsh wiki](https://github.com/robbyrussell/oh-my-zsh/wiki)

[github readme](https://github.com/robbyrussell/oh-my-zsh/wiki/Installing-ZSH)

***END***
