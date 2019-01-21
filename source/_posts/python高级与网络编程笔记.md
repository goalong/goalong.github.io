---
title: python高级与网络编程笔记
date: 2018-05-14
tags: 
	- python 
	- 网络

---


读https://aceld.gitbooks.io/python/content/ 这本电子书的笔记，正好有许多我感兴趣的内容，感谢作者。

### 生成器 迭代器
G = ( x*2 for x in range(5)) 生成器，把列表推倒的中括号换成小括号就成了生成器，可以for 循环遍历，也可以调用next函数来遍历

生成器优点：节约内存；每一次调用所使用的参数是上一次时保留的

可以被next()函数调用并不断返回下一个值的对象称为迭代器：Iterator
凡是可作用于 for 循环的对象都是 Iterable 类型

集合数据类型如 list 、 dict 、 str 等是 Iterable 但不是 Iterator ，不过可以通过 iter() 函数获得一个 Iterator 对象

### 闭包
在函数内部再定义一个函数，并且这个函数用到了外边函数的变量，那么称里面的这个函数为闭包

### 装饰器
	@w1
	def f1():
	    print('f1')
可以理解为f1 = w1(f1), 即原来的函数作为参数传给装饰器w1, 装饰器返回的值赋值给f1

用途： 记录日志， 函数执行时间统计， 函数执行前预备处理及执行后清理， 权限校验， 缓存等

	def timefun(func):
	    def wrappedfunc(*args, **kwargs):
	        print("%s called at %s"%(func.__name__, ctime()))
	        func(*args, **kwargs)
	    return wrappedfunc
	@timefun
	def foo(a, b, c):
	    print(a+b+c)
	    
被装饰的函数foo此处即相当于成为了函数wrappedfunc, 因此wrappedfunc的参数设置的跟foo一样即可，当然上面的写法用了不定长参数可以处理任意参数的搭配。被装饰后，foo的返回值即wrappedfunc的返回值。

	def timefun_arg(pre="hello"):
	    def timefun(func):
	        def wrappedfunc():
	            print("%s called at %s %s"%(func.__name__, ctime(), pre))
	            return func()
	        return wrappedfunc
	    return timefun

	@timefun_arg("itcast")
	def foo():
	    print("I am foo")
	    
装饰器带参数，向正常函数带参数一样，此时foo = timefun_arg("itcast")(foo), 注意此时装饰器的实现是嵌套了三层的函数

	class Itcast(object): 
	    def __init__(self, func): 
	        print("--初始化--")
	        self._func = func 
	
	    def __call__(self):  
	        print('--装饰器中的功能--')
	        self._func() 
类装饰器通过重载__call__()来实现。

### import
sys.path 返回搜索的路径

避免循环导入： 代码结构上将共同引用的放在一起； 函数里导入

### 私有变量
下划线开头的变量在from XXX import * 时不会导入，继承时子类也不继承该变量。

	class Money(object):
	    def __init__(self):
	        self.__money = 0
	
	    @property
	    def money(self):
	        return self.__money
	
	    @money.setter
	    def money(self, value):
	        if isinstance(value, int):
	            self.__money = value
	        else:
	            print("error:不是整型数字")
	            
property方法实现一个属性的设置和读取方法,可做边界判定

### 垃圾回收
python采用的是引用计数机制为主，标记-清除和分代收集两种机制为辅的策略
todo, 细看 https://aceld.gitbooks.io/python/content/516d29-python-qi-ta-zhi-shi-dian/10-la-ji-hui-shou/la-ji-hui-653628-4e8c29.html

小整数对象池，范围-5～257，避免频繁重复申请和销毁内存空间
单个字符也是共用，同样的字符串，其中不包含空格， 也是共用

每个对象最基础的结构就包括两项，一个是类型，一个是引用计数
引用计数的优点：
简单 实时
缺点：
维护引用计数消耗资源 循环引用

触发垃圾回收的情况：
1.  调用gc.collect()
2. gc模块的计数器达到阈值
3. 程序退出时

### 进程
程序是编写的代码，进程是运行中的代码以及运行环境

创建进程

import os
pid = os.fork()  # fork只能在*nix系统上执行
if pid == 0:
	print("this is 1")
else:
	print("this is 2")
	
fork执行时，操作系统会创建一个新的子进程，然后复制父进程的所有信息到子进程中，子进程和父进程都会得到一个返回值pid, 对父进程来说这个pid是子进程的进程ID，对子进程来说是0. 这样做的理由是，一个父进程可以fork出很多子进程，所以，父进程要记下每个子进程的ID，而子进程只需要调用getppid()就可以拿到父进程的ID.

fork这个系统调用比较特殊，普通函数调用一次返回一次，fork调用一次返回两次

getpid 返回当前进程的进程ID， getppid 返回父进程的进程ID
	

