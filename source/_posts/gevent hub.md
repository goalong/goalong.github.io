---
title: [译]gevent的hub
date: 2019-03-09
tags: 
	- gevent
---


hub是gevent中非常重要的一部分，但是在官方文档对它的描述中，除了宏观的描述了一下是一个事件循环外，就没怎么具体的说了，关于它的目的、它是什么以及它如何工作的都没怎么叙述。这篇文章就是为了改变这个现状以及回答如下的一些问题：

* hub是什么
* hub是如何运行事件循环的
* greenlet是如何进行IO的？如何一边运行着事件循环，一边又运行着其他的greenlet，并且在他们阻塞的时候选择一个正确的greenlet去执行的？

### hub是什么
先从官方文档上的描述说起吧：

```
当gevent API中的一个函数A要阻塞的时候，它将获得一个gevent.hub.Hub的实例（Hub实
例是一个非常特别的运行着事件循环的greenlet），然后函数A的那个greenlet被切换到
Hub(将控制权交还给Hub）。如果当时还没有gevent.hub.Hub实例，就会自动创建一个。
```

这个描述好像还是不够清晰，API文档中对hub确实也没怎么细说。

为了弄明白这到底什么意思, 咱们分成几小块细细说来。

### hub其实是一个greenlet
首先，hub本身也是一个greenlet。greenlet是绿色线程的一个实现。他们看起来像一个常规的操作系统级别的线程，每个greenlet都有一个调用栈（Python解释器的C调用栈加上Python调用栈), 而且也代表着程序中的一段控制流。不同之处在于多个greenlet可以共存在一个线程中，但是任意时刻一个线程中只有一个greenlet在运行。

另一个greenlet与操作系统线程不同的地方在于它们是协作式调度的，而不是抢占式调度。这意味着为了让另一个greenlet运行，当前运行的greenlet必须放弃控制权，这叫做切换。而且，切换时必须明确选择要让哪个greenlet来接替执行。需要在当前的greenlet里调用目的地greenlet的switch方法来进行这样的切换。当切换发生时，当前greenlet的调用栈会被保存下来，目的地greenlet的调用栈被放到正确的位置，然后执行流在目的地greenlet上次切换的地方继续执行。

### hub运行着事件循环
好的，现在我们知道了hub是一个greenlet，是一种协作式调度的调用栈或者说执行线程，而且这个执行线程运行着事件循环。

下面是gevent文档里对事件循环的描述：

```
gevent让操作系统传发出事件让事件循环知道何时一个事件发生了，例如数据已经传输过来了
socket可以读了，从而代替阻塞和一直等待socket操作完成（轮询）的方式。通过这样做，
gevent可以继续运行下一个或许已经有事件就绪的greenlet。这样重复的注册事件以及在事件
发生时进行处理的流程就是事件循环了。
```

用图来表示的话，一个事件循环就像下面这样（这是个libuv事件循环比较宏观的概览，不要在意具体的细节和技术）：

![loop_iteration.png](https://i.loli.net/2019/03/10/5c84fa87e414a.png)

运行事件循环意味着hub从那个循环的顶部开始进入，然后开始一圈圈的无限循环。gevent支持的事件循环是由C实现的，因此从循环的顶部进入意味着进行一个C的函数调用，那个C函数调用是不会返回的（因为是一个无限循环）。

注册事件是由叫做watchers的对象完成的。每个watcher对象等待着一个事件的发生。

但是其他的实际进行读写socket的greenlet是如何开始运行的呢？真正有趣的地方开始了。

### 冲出hub
再看一下上面的图示，注意以Call开头的，例如"Call pending callbacks"，以及以Run开头的，例如"Run due timers"，这些部分正是仍然在C函数中的事件循环给gevent机会去执行一些例如处理IO事件或者处理预先注册的事件（timer)的时候。

gevent对于每一个它咬处理的事件都创建了watcher然后把watcher交给事件循环。每个watcher都有一个回调函数，当预期的事件发生时事件循环会调用这个回调函数。（把watcher交给事件循环并且给它绑定一个回调函数被称为开始一个watcher). 回调函数几乎什么都能做。如果恰好回调函数里调用了g.switch(), 呜呼，突然之间，hub以及正在运行的事件循环被暂停了（C调用栈也被保存了），然后某个其他的greenlet开始接替进行执行。

### 我们是如何第一次进入hub的
我们刚看到了执行流如何离开正在热火朝天的运行着事件循环的hub的：某个回调函数里面调用了greenlet.switch方法。但是我们最开始的时候又是怎么在hub里运行事件循环的呢？

我们再回头看一下文档对hub的描述：

```
当gevent API中的一个函数A要阻塞的时候，它将获得一个gevent.hub.Hub的实例（Hub实
例是一个非常特别的运行着事件循环的greenlet），然后函数A的那个greenlet被切换到
Hub(将控制权交还给Hub）。如果当时还没有gevent.hub.Hub实例，就会自动创建一个。
```

所以gevent中任何会阻塞的函数（例如gevent.socket.socket.read()）都会将执行切换到hub。如果是第一次切换到hub里，hub就会启动事件循环，否则，如果hub已经在运行着事件循环了，就会从上次离开的地方开始然后对下一个事件（IO, timer）通过回调函数进行处理，如果没有事件就继续等待下一个事件发生。

你可以想象成一个乒乓球一样的场景，hub在桌子的一边，其他所有的greenlet在另一边。hub通过网络将球打到对面的greenlet，greenlet再把球打回来，然后hub把球打过去给另一个greenlet，就这样无限循环。

### 写点代码

现在我们拥有了一些背景知识，可以写一些代码看看它们是如何组合在一起工作的了。

gevent大多数的阻塞的函数都会在它们的实现里调用Hub.wait().下面是gevent.socket.socket.recv()大致的流程：

```
def recv(self, *args):
    while True:
        try:
            return _socket.socket.recv(self._sock, *args)
        except error as ex:
            if ex.args[0] != EWOULDBLOCK:
                raise
        self.hub.wait(self._read_event)
```

这段代码一直循环知道能读到数据。如果我们尝试读取数据，结果发现没有，就会调用hub.wait()把执行切换到事件循环，这将跳转回到hub，确保事件循环监视着这些socket, 然后继续运行。

Hub.wait大致的实现是下面这样（这是简化的例子，真实情况要比这安全而且使用了[Waiter](http://www.gevent.org/api/gevent.hub.html#gevent.hub.Waiter):

```
def wait(self, watcher): # Hub.wait()
    # `watcher` is an event-loop object with a callback. When the
    # event it's waiting for happens, its callback gets called.
    current_greenlet = getcurrent()
    # The callback for this event will switch back to
    # this current greenlet
    watcher.start(current_greenlet.switch) # Ask the event loop to watch this
    try:
        # Start running the hub.
        self.switch()
        # Once we get here, it's because the watcher's callback
        # fired and this greenlet got switched into.
        return # Let the blocking code continue, its event is ready
    finally:
        watcher.stop()
```

### 一些注意事项
* 一个greenlet执行完成后会发生什么？
	控制将回到它的parent greenlet. gevent让hub作为所有它运行的gevent的parent，因此当一个parent不论是成功完成执行还是有未捕获的异常而意外终止，执行流都会回到hub，以便继续运行事件循环。

* 当我们想开始一个新的greenlet时会发生什么？
	gevent.spawn()创建一个新的greenlet，并且安排它在事件循环的下一轮循环时开始运行。猜想一下它是如何实现的，它创建一个事件的watcher，并且绑定一个new_greenlet.switch()的回调函数。这种事件的watcher是一个"prepare" watcher，这种类型的事件会在每一轮循环的开始就变的就绪。
	
* gevent的锁和超时机制是如何实现的？
	你得等到下一篇文章来探讨这些主题了。
	
原文：

[gevent's hub](https://dev.nextthought.com/blog/2018/05/gevent-hub.html)

注：原文的作者Jason Madden也是目前gevent库的主要维护者。



	
