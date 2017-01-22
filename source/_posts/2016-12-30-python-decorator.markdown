---
layout: post
title:  "python系列--python decorator 简 记"
date:   2016-12-30 14:00:25
tag: python
---

## 目录结构：

[装饰器的本质](#A)

[装饰器的几种形式](#B)


注：以下内容皆出自 酷壳 [PYTHON修饰器的函数式编程](http://coolshell.cn/articles/11265.html) ,本文只是便于自己记忆，做的一篇简单记录。

<a name="A"></a>

## 装饰器的本质

看下一个最简单的装饰器的例子：

```python
def hello(fn):
    def wrapper():
        print "hello, %s" % fn.__name__
        fn()
        print "goodby, %s" % fn.__name__
    return wrapper
 
@hello
def foo():
    print "i am foo"
 
foo()

```

@注解 是一个Python 中的语法糖，比如一个如下语句
```python
@decorator
def func():
    pass
```

它会被解释为 ： 
```python
  func = decorator(func)  
```

其实就是把一个函数当作参数传到另一个函数，然后再回调，最后再把decorator这个函数的返回值赋值回了原来的func。

落实到刚开始的例子就是：
```python
def hello(fn):
    def wrapper():
        print "hello, %s" % fn.__name__
        fn()
        print "goodby, %s" % fn.__name__
    return wrapper

def foo():
    print "i am foo"

foo = hello(foo)

foo()
```

OK,这样就理清了！

<a name="B"></a>

## 装饰器的几种形式

- 带参数的形式：

```python
def makeHtmlTag(tag, *args, **kwds):
    def real_decorator(fn):
        css_class = " class='{0}'".format(kwds["css_class"]) \
                                     if "css_class" in kwds else ""
        def wrapped(*args, **kwds):
            return "<"+tag+css_class+">" + fn(*args, **kwds) + "</"+tag+">"
        return wrapped
    return real_decorator
 
@makeHtmlTag(tag="b", css_class="bold_css")
@makeHtmlTag(tag="i", css_class="italic_css")
def hello():
    return "hello world"
 
print hello()
# 输出：
# <b class='bold_css'><i class='italic_css'>hello world</i></b>
```

为了让 hello = makeHtmlTag(arg1, arg2)(hello) 成功，makeHtmlTag 必需返回一个decorator（这就是为什么我们在makeHtmlTag中加入了real_decorator()的原因），这样一来，我们就可以进入到 decorator 的逻辑中去了—— decorator得返回一个wrapper，wrapper里回调hello。



- 类的形式

```python
class myDecorator(object):
 
    def __init__(self, fn):
        print "inside myDecorator.__init__()"
        self.fn = fn
 
    def __call__(self):
        self.fn()
        print "inside myDecorator.__call__()"
 
@myDecorator
def aFunction():
    print "inside aFunction()"
 
print "Finished decorating aFunction()"
 
aFunction()
 
# 输出：
# inside myDecorator.__init__()
# Finished decorating aFunction()
# inside aFunction()
# inside myDecorator.__call__()
```

可以看出在构造装饰器装饰的函数时，执行__init__函数，在真正调用aFunction()函数时会调用__call__。

看下重写上面的html.py的代码：

```python
class makeHtmlTagClass(object):
 
    def __init__(self, tag, css_class=""):
        self._tag = tag
        self._css_class = " class='{0}'".format(css_class) \
                                       if css_class !="" else ""
 
    def __call__(self, fn):
        def wrapped(*args, **kwargs):
            return "<" + self._tag + self._css_class+">"  \
                       + fn(*args, **kwargs) + "</" + self._tag + ">"
        return wrapped
 
@makeHtmlTagClass(tag="b", css_class="bold_css")
@makeHtmlTagClass(tag="i", css_class="italic_css")
def hello(name):
    return "Hello, {}".format(name)

```

注意参数的调用，如果decorator有参数的话，__init__()成员就不能传入fn了，而fn是在__call__的时候传入的。

被decorator的函数其实已经是另外一个函数了，对于最前面那个hello.py的例子来说，如果你查询一下foo.__name__的话，你会发现其输出的是“wrapper”，而不是我们期望的“foo”，这会给我们的程序埋一些坑。所以，Python的functool包中提供了一个叫wrap的decorator来消除这样的副作用。所以就有了下面这种形式：

```python
from functools import wraps
def hello(fn):
    @wraps(fn)
    def wrapper():
        print "hello, %s" % fn.__name__
        fn()
        print "goodby, %s" % fn.__name__
    return wrapper
 
@hello
def foo():
    '''foo help doc'''
    print "i am foo"
    pass
 
foo()
print foo.__name__ #输出 foo
print foo.__doc__  #输出 foo help doc
```



## 参考文章

[PYTHON修饰器的函数式编程](http://coolshell.cn/articles/11265.html)




***END***
