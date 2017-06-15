---
layout: post
title:  "python-- python 中的一些隐藏特性"
date:   2017-06-14 14:21:32
tags: 
  - python
  - hidden feature
---



## 引出

偶然间翻到Stackoverflow上的[这个问题](https://stackoverflow.com/questions/101268/hidden-features-of-python#111176)，浏览了一下，觉得有点意思，摘录了一些感觉能用到的纪录在此。

## 使用‘ * ** ’ 代表函数的list/dict参数

```python
def draw_point(x, y):
    print(x,y)
point_foo = [3, 4] # or (3,4)
point_bar = {'y': 3, 'x': 2}

draw_point(*point_foo)
draw_point(**point_bar)
```

这个特性其实使用还算常见，不过pylint不建议这样使用，因为debug的时候会不清晰。

## 链式比较操作

```bash
>>> x = 5
>>> 1 < x < 10
True
>>> 10 < x < 20 
False
>>> x < 10 < x*10 < 100
True
>>> 10 > x <= 9
True
>>> 5 == x > 4 
True
```
比如最后一个比较，python会将这一句转化为 (5 == x and x > 4) ，这是跟C或java不一样的。 

## 装饰器

装饰器用一个函数或方法封装另一个函数以增加功能或修改参数，结果等。

```bash
>>> def print_args(function):
>>>     def wrapper(*args, **kwargs):
>>>         print 'Arguments:', args, kwargs
>>>         return function(*args, **kwargs)
>>>     return wrapper

>>> @print_args
>>> def write(text):
>>>     print text

>>> write('foo')
Arguments: ('foo',) {}
foo
```

## 小心使用可变的默认参数

```bash
>>> def foo(x=[]):
...     x.append(1)
...     print x
... 
>>> foo()
[1]
>>> foo()
[1, 1]
>>> foo()
[1, 1, 1]
```

可以加一个哨兵值来做判断：

```bash
>>> def foo(x=None):
...     if x is None:
...         x = []
...     x.append(1)
...     print x
>>> foo()
[1]
>>> foo()
[1]
```

## 利用python 正则语法树来debug正则表达式

```bash
>>> re.compile("^\[font(?:=(?P<size>[-+][0-9]{1,2}))?\](.*?)[/font]",
    re.DEBUG)
at at_beginning
literal 91
literal 102
literal 111
literal 110
literal 116
max_repeat 0 1
  subpattern None
    literal 61
    subpattern 1
      in
        literal 45
        literal 43
      max_repeat 1 2
        in
          range (48, 57)
literal 93
subpattern 2
  min_repeat 0 65535
    any None
in
  literal 47
  literal 102
  literal 111
  literal 110
  literal 116

``` 
当然，最常用的还是使用注解的方式以增强正则的可读性。

```bash
>>> re.compile("""
 ^              # start of a line
 \[font         # the font tag
 (?:=(?P<size>  # optional [font=+size]
 [-+][0-9]{1,2} # size specification
 ))?
 \]             # end of tag
 (.*?)          # text between the tags
 \[/font\]      # end of the tag
 """, re.DEBUG|re.VERBOSE|re.DOTALL)
```

## 利用enumerate 获取list的下标

```bash
>>> a = ['a', 'b', 'c', 'd', 'e']
>>> for index, item in enumerate(a, start=1): print index, item
...
1 a
2 b
3 c
4 d
5 e
>>>
```

## 生成器

```bash
>>> n = ((a,b) for a in range(0,2) for b in range(4,6))
>>> for i in n:
...   print i 

(0, 4)
(0, 5)
(1, 4)
(1, 5)
```
需要注意的是，列表生成式是实实在在生成了列表存在了内存里，可以多次迭代使用，而生成器只能迭代一次。

## 巧用list中的slice操作

slice操作如下：

```bash
a = [1,2,3,4,5]
>>> a[::2]  # iterate over the whole list in 2-increments
[1,3,5]
```
将一个list逆序，最简洁写法：

```bash
>>> a[::-1]
[5,4,3,2,1]
```

## for...else 语法

```python
for i in foo:
    if i == 0:
        break
else:
    print("i was never 0")
```
上述代码如果break，那么else代码是不会执行的。上述代码的执行顺如跟下面的代码是一样的：
```python
found = False
for i in foo:
    if i == 0:
        found = True
        break
if not found: 
    print("i was never 0")
```

## 两个变量互换

```bash
>>> a = 10
>>> b = 5
>>> a, b
(10, 5)

>>> a, b = b, a
>>> a, b
(5, 10)
```
## with 声明语句

with 在Python2.5需要从_future_模块导入，2.6+已经可以直接使用。示例：

```python
from __future__ import with_statement

with open('foo.txt', 'w') as f:
    f.write('hello!')
```
with 语句在这里的作用是在调用with时，会调用这个文件实例的 __enter__ 方法，和 __exit__ 方法，如果发生异常，异常的详细信息会传给 __exit__ 方法处理并抛出，with在这里的主要用途是保证文件句柄最终关闭，其实with 可以看作是异常处理的一种抽象。
除了打开文件经常使用with外，线程锁以及数据库事务也经常用到。


## 参考文章


[Hidden features of Python ](https://stackoverflow.com/questions/101268/hidden-features-of-python#113198)


***END***
