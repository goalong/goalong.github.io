---
title: Python装饰器总结
date: 2019-02-28
tags: 
	- Python
---


也写过一些装饰器，但记忆不够深刻，这次系统总结一下以备忘。

### 功能
给一个函数增加额外的功能或操作

### 实现

一个装饰器就是一个函数或类，它接受一个函数作为参数并返回一个新的函数，返回的函数跟被装饰的函数接收相同的参数

### @语法糖

	@d
	def f ():
    	pass
    	
    上面的下面这样是等价的。	
    def f ():
    	pass
    f = d(f) 

在过往的工作和学习中，遇到的使用装饰器的场景有以下几种：

* 权限校验，比如某个接口需要用户是登录的状态才能访问
* 日志记录，记录接口的调用和执行信息，参数、执行时间等
* 异常处理，例如sentry收集异常

### 简单装饰器
```pyhton
def deco1(f):
	def wrapper(*args, **kw):
    	 print(f.__name__)
    	 return f(*args, **kw)
    return wrapper
    
@deco1    
def foo(name):
	print("hello, {}".format(name))
```
装饰器没有参数，被装饰的函数可以接收参数

### 带参数的装饰器

```pyhton
def deco2(flag):
    def helper(f):
        def wrapper(*args, **kw):
            if flag:
                print("func name:", f.__name__)
            return f(*args, **kw)
        return wrapper
    return helper


@deco2(True)
def bar(name):
    print(name)

bar("test")
```

装饰器也带参数, 注意嵌套了两层。

### 装饰器的副作用

你写了一个装饰器作用在某个函数上，但是这个函数的重要的元信息比如名字、文档字符串、注解和参数签名都丢失了，应该使用 functools 库中的 @wraps 装饰器来注解底层包装函数, wraps本身也是一个装饰器，它能把原函数的元信息拷贝到装饰器里面的函数中，这使得装饰器里面的函数也有和原函数一样的元信息了.

```pyhton
from functools import wraps
def deco1(f):
	@wraps(f)
	def wrapper(*args, **kw):
    	 print(f.__name__)
    	 return f(*args, **kw)
    return wrapper
```

### 多个装饰器

```
@a
@b
@c
def f ():
    pass
```

等价于 f = a(b(c(f)))), 先调用最里层的，然后从里到外。

### 类装饰器

依靠__call__方法实现

```python
class Foo(object):
    def __init__(self, func):
        self._func = func

    def __call__(self):
        print ('class decorator runing')
        self._func()
        print ('class decorator ending')

@Foo
def bar():
    print ('bar')

bar()
```

参考：

[Python Cookbook](https://python3-cookbook.readthedocs.io/zh_CN/latest/chapters/p09_meta_programming.html)

[理解 Python 装饰器看这一篇就够了](https://foofish.net/python-decorator.html)





    