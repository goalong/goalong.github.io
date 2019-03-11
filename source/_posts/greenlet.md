---
title: greenlet笔记
date: 2019-02-16
tags: 
	- Python
---


### 简介
greenlet, 是一种微型线程，也叫协程。基于生成器的协程在进行协程之间的切换时只能由当前的协程切换到其调用者，而greenlet提供的切换允许你切换到指定的协程。

```python
from greenlet import greenlet

def test1():
    print(12)
    gr2.switch()
    print(34)

def test2():
    print(56)
    gr1.switch()
    print(78)

gr1 = greenlet(test1)
gr2 = greenlet(test2)
gr1.switch()
print("finish")
```
使用greenlet(func)创建一个greenlet, 最后一行gr1.switch()将程序切换到test1函数进行执行，输出12，然后gr2.switch()切换到test2运行，test1此时被暂停，test2输出56，接着test2中又调用了gr1.switch()，这样回到刚才暂停的test1继续执行，输出34，然后test1执行完毕，gr1也就死亡了，这时执行回到最初的gr1.switch(),输出finish，全部执行完毕，进程结束。注意test2的78是不会被输出的。

### parents
一个greenlet的执行结束了就被称为死亡（dead)了，遇到未能捕捉的异常也会导致greenlet的死亡。greenlet死亡之后程序往哪执行呢？答案是它的parent greenlet。每个greenlet都有一个parent greenlet，默认是该greenlet的创建者。greenlet死亡之后会切换到其parent greenlet继续执行。可以说greenlet的组织类似一棵树，最外层的不是在greenlet中创建的greenlet, 它的parent默认是main greenlet, 也就是树的根。

上面的例子中，gr1和gr2的parent都是main greenlet，其中一个死了，执行就会回到main greenlet。未捕获的异常也会展开至parent greenlet。

### greenlet类型
greenlet.greenlet是greenlet的类型，主要支持如下操作：

* greenlet(run=None, parent=None)

	创建一个greenlet实例，run是一个可调用对象，parent是指定的parent greenlet, 默认是当前的greenlet。
	
* g.getcurrent()
	返回当前的greenlet，也就是调用这个函数的greenlet
	
* g.GreenletExit
	这个特殊的异常被触发后，即使不处理，也不会被抛给parent greenlet。
* g.run
	可调用对象，开始运行g的时候会执行这个可调用对象，执行完毕后这个属性不再存在
* g.parent
	当前greenlet的parent greenlet，可以修改，但几个greenlet的parent greenlet指向不能形成闭环。
* g.dead
	死了的话值为True
* g.throw
	切换到指定greenlet后立即抛出异常

		
greenlet类型支持继承，继承它的类需要实现自己的run方法。
	
### 切换
切换在switch调用时发生，执行会被切换到switch方法的调用者，如果一个greenlet死了，执行会切换到其parent greenlet, 调用switch时可以传递一个对象或者异常给调用者，这是一个很方便的在greenlet之间进行传递消息的方式。可以看这个例子：

```python
def test1(x, y):
    z = gr2.switch(x+y)
    print(z)

def test2(u):
    print(u)
    gr1.switch(42)

gr1 = greenlet(test1)
gr2 = greenlet(test2)
gr1.switch("hello", " world")
# 输出
hello world
42
```
注意传递给test1()和test2()的参数并不是greenlet创建时提供的，而是第一次调用switch()时传递的。

* g.switch(*args, **kwargs)
	将执行切换到g, 发送给它特定的参数，如果g还没有开始运行，则现在开始运行。
	
如果一个greenlet完成运行，其返回值会被返回给其parent。切换到一个死了的greenlet则会被切换给其parent，或者parent的parent，以此类推，直到main greenlet。

如果greenlet g2中调用x = g1.switch(y)，意思是切换到g1,并向它传递参数y，后面继续执行的时候某个greenlet如果切换到g2, 会把其传的参数赋值给x.

### 值得注意的地方
* greenlet必须保证执行完毕，如果switch出去之后没有switch回来，一直保持挂起状态，会造成内存泄漏
* greenlet可以和多线程混合使用，但此时每个线程都有自己的main greenlet和对应的子greenlet, 不同线程之间的greenlet无法相互切换

日常的开发中，很少直接用到这个库，一般是使用在它基础上开发的库，例如gevent, 不过了解一下其工作原理还是对构筑在其基础上的软件的理解能起到帮助作用的。


参考：

[greenlet官方文档](https://greenlet.readthedocs.io/en/latest/#)

[greenlet 详解](https://www.cnblogs.com/xybaby/p/6337944.html)

	
	
	
