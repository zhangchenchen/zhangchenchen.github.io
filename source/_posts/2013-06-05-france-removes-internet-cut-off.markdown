---
layout: post
title:  "mysql 系列--Invalid default value for UPDATE_TIME"
date:   2013-06-05 20:33:11
tags: mysql
---


导入SQL文件时出现“Invalid default value for UPDATE_TIME”,UPDATE_TIME 是timeStamp类型。设置的默认值为“00-00-00 0000 0000” 。

经过不断试验发现是因为sql_mode 的设置原因，按照网上说的更改my_default.ini 并重启mysql后发现还是不行。
SELECT @@global.sql_mode;
SELECT @@session.sql_mode;

sql_mode 依然没有改变

后直接在数据库层面改：

 >SET GLOBAL sql_mode='';
  SET SESSION sql_mode='';

 将sql_mode 设置为空，问题解决。
 关于sql_mode ，[见此](http://dev.mysql.com/doc/refman/5.5/en/sql-mode.html#sql-mode-important)



  ***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***