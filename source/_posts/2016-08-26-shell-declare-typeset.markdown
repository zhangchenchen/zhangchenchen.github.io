---
layout: post
title:  "shell脚本系列--declare 与 typeset"
date:   2016-08-26 09:24:25
tags: shell
---

## 目录结构：

[命令用途 ](#A)

[常用命令参数 ](#B)

[使用示例](#C)





<a name="A"></a>

## 命令用途 

declare 与 typeset 命令是bash的内建命令，两者是完全一样的，用来声明shell变量，设置变量的属性。


<a name="B"></a>

## 常用命令参数

 1. -r  设置变量为只读

 2. -i 设置变量为整数

 3. -a 设置变量为数组array

 4. -f 如果后面没有参数的话会列出之前脚本定义的所有函数，如果有参数的话列出以参数命名的函数

 5. -x 设置变量在脚本外也可以访问到



<a name="C"></a>

## 使用示例

```bash
#!/bin/bash

func1 ()
{
  echo This is a function.
}

declare -f        # Lists the function above.

echo

declare -i var1   # var1 is an integer.
var1=2367
echo "var1 declared as $var1"
var1=var1+1       # Integer declaration eliminates the need for 'let'.
echo "var1 incremented by 1 is $var1."
# Attempt to change variable declared as integer.
echo "Attempting to change var1 to floating point value, 2367.1."
var1=2367.1       # Results in error message, with no change to variable.
echo "var1 is still $var1"

echo

declare -r var2=13.36         # 'declare' permits setting a variable property
                              #+ and simultaneously assigning it a value.
echo "var2 declared as $var2" # Attempt to change readonly variable.
var2=13.37                    # Generates error message, and exit from script.

echo "var2 is still $var2"    # This line will not execute.

exit 0                        # Script will not exit here.
```



## 参考文章

[Advanced Bash-Scripting Guide](http://tldp.org/LDP/abs/html/declareref.html)

[linux bash shell之declare ](http://www.cnblogs.com/fhefh/archive/2011/04/22/2024857.html)


***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***
