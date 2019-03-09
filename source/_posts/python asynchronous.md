---
title: [译文]Python异步之旅
date: 2019-03-04
tags: 
	- Python
---

一般的程序都是一行接着一行的执行的，例如，你要请求远端的服务器上的一份资源，那就意味着在资源返回之前中你的程序什么也没法做，只能在那干等着，等到响应返回才能继续后续的操作。有些情况下，这种等待可以接受，而有些情况下则无法接受。

### 线程
![屏幕快照 2019-03-09 下午10.05.10.png](https://i.loli.net/2019/03/09/5c83c83cec040.png)
对这种情景的标准处理办法就是使用线程，一个进程可以创建多个线程，各个线程在同一时刻处理自各的操作。这些线程允许你的程序在同一时刻处理多个事情。多线程当然也有弊端。多线程的程序逻辑将会更加复杂，也更容易出错，使用多线程有一些普遍的问题如下：竞态限制、死锁、活锁(livelock)与饿死(starvation)。

### 上下文切换
异步编程可以避免这些问题的产生，然而它最初是为解决另一个问题而设计的：CPU上下文切换。当你有多个线程在运行的时候，每个CPU核依然每次只能运行一个线程。为了让所有的线程和进程能共享CPU资源，CPU就一直不停的切换上下文。简单来说，CPU保存了所有线程的上下文，然后不停的随机在线程之间切换。而且线程也不是免费的，也是一种资源。

异步编程基本上就是使用一种用户态的线程，依靠应用程序自身而不是CPU对其进行上下文切换。在异步编程的世界，上下文切换只是在程序指定的地方进行，而不是像CPU那样完全靠随机切换。

### 超级高效的秘书

我们用一个非计算机领域的例子来类比前面的一些概念。想象我们有一个超级高效的秘书，总能把事情搞定而且一点时间也不浪费，总想着能最大化每一秒的效率。我们叫这个秘书Bob吧。Bob要负责五件事：回复电话、接待来访的客人、打算预定一个机票、处理会议日程表以及归档文件。我们想象一下不怎么忙的时候，来电、客人、以及会议都很少，Bob大部分时间一边归档文件一边在电话里跟航空公司预订机票。当有来电时，他暂时放下订机票的电话，回复来电，处理完之后继续订票的电话交谈。任何时刻一个事件发生的时候，归档文件总会被放下等待稍后处理，因为那不需要立即就处理。这是一个人在同一时刻处理多个任务，在合适的时机进行上下文切换的例子。这么来说的话Bob也是异步的了。

![屏幕快照 2019-03-09 下午10.03.52.png](https://i.loli.net/2019/03/09/5c83c83d2e4b1.png)

多线程版本的Bob看起来会像是有五个Bob，每个只负责处理一件事，但是任意一个时刻只能有其中一个人被允许工作。这需要一个设备来控制允许哪个Bob工作，这个设备对这五个任务一无所知。因为这个设备不了解这几个任务的特性，它将不停的在这几个Bob之间随机切换，即使可能其中三个Bob明明只是坐在那啥也不干。举例来说，负责归档文件的Bob在做着自己的工作呢突然被暂停了，现在轮到接听来电的Bob工作了，然而此时并没有来电，所以他干脆去睡觉了。有许多时间被这些无意义的切换浪费掉了，大概57%的上下文切换是毫无意义的。尽管CPU上下文切换速度很快，但这并不是免费的。

### 绿色线程
绿色线程是比较原始的异步编程方式。绿色线程看起来像普通线程，但是它是应用程序代码进行的调度，而不是操作系统。Gevent是一个著名的使用绿色线程的Python库。Gevent基本上就是绿色线程加eventlet，是一个非阻塞IO的网络库，它通过给普通的Python库打猴子补丁，使其拥有非阻塞IO能力，下面是一个使用Gevent一次对多个URL发送请求的例子。

```
import gevent.monkey
from urllib.request import urlopen
gevent.monkey.patch_all()
urls = ['http://www.google.com', 'http://www.yandex.ru', 'http://www.python.org']

def print_head(url):
    print('Starting {}'.format(url))
    data = urlopen(url).read()
    print('{}: {} bytes: {}'.format(url, len(data), data))

jobs = [gevent.spawn(print_head, _url) for _url in urls]

gevent.wait(jobs)
```

可以看到，Gevent的接口看起来和threading的差不多。然而在底层，它是使用协程而不是真正的线程，并通过一个事件循环(event loop)来进行调度。这意味着你不需要理解协程就可以收到轻量级线程的好处，但仍然要面临多线程要面临的那些问题。Gevent对于那些已经理解了多线程但想要轻量级线程的人来说是一个好的选择。

让我们明确一下异步编程到底是怎么工作的。一个很普遍的进行异步编程的方式是使用一个事件循环。这个事件循环就像它的名字描述的那样，有一个事件的队列和一个循环，循环会不断的从队列中取出事件进行处理。这些事件被称作协程，是一段小的指令集。

### 回调风格的异步
在Python众多的异步库中，最流行的当属Tornado和gevent了。既然已经说过gevent，现在来谈一谈Tornado是怎么工作的吧。Tornado是一个异步的网络框架，使用回调的方式进行异步网络IO。回调是一种函数，意味着在完成的时候执行这个函数，基本上就是一个当一个工作完成的时候再执行的钩子。就像你打电话给客服，留下你的手机号然后挂断，以便客服有空的时候给你回电，而不是你一直拿着电话等客服有空。

我们来看一下怎么用tornado做上面的同样的事。

```
import tornado.ioloop
from tornado.httpclient import AsyncHTTPClient
urls = ['http://www.google.com', 'http://www.yandex.ru', 'http://www.python.org']

def handle_response(response):
    if response.error:
        print("Error:", response.error)
    else:
        url = response.request.url
        data = response.body
        print('{}: {} bytes: {}'.format(url, len(data), data))

http_client = AsyncHTTPClient()
for url in urls:
    http_client.fetch(url, handle_response)
    
tornado.ioloop.IOLoop.instance().start()
```

简单介绍一下上面的代码，倒数第二行是调用一个tornado的方法叫做AsyncHTTPClient.fetch，这个方法会以非阻塞的方式请求一个URL，基本上就是发出请求然后不等待响应便立即返回，以便程序在这段等待的时间可以处理其他的事。因为还没等响应返回就已经去执行后面的代码了，所以也没办法获取这个方法的返回值。解决这个问题的办法就是使用回调函数，等响应返回时让回调函数来处理，上面代码中的回调函数就是handle_response。

注意handle_response这个回调函数，一开始是检查错误，这是非常有必要的，因为在这里不能抛出异常，一旦抛出异常，由于事件循环的原因，也将不是合适的代码去处理异常。当fetch方法执行时，它发起http请求，然后把处理响应的工作交给了事件循环。等我们注意到有错误或异常发生时，调用栈上只剩事件循环和这个回调函数了，都没法对异常进行处理。因此任何被抛给回调函数的异常都会导致事件循环和整个程序崩溃。所以所有的错误会被当作对象进行传递而不是被抛出，这意味着如果你忘了检查错误，那这些错误就会被抛弃而丢失了。

在异步的世界里，回调的另一个问题就是不让操作阻塞的唯一办法就是使用回调，这会导致很长的回调调用链，一个回调接着一个回调接着一个回调。
![屏幕快照 2019-03-09 下午10.05.40.png](https://i.loli.net/2019/03/09/5c83c836d65d5.png)

### 对比
如果要防止IO阻塞，要么使用多线程要么使用异步。多线程带来的问题有死锁、竞态条件、资源饿死等。此外它还有大量的CPU上下文切换开销。异步编程能解决上下文开销问题，但也带来了新的问题，在python中异步编程的方式有绿色线程和回调风格这两种。

#### 绿色线程风格
* 线程由应用程序级别来管理，而不是操作系统
* API与多线程相似，对熟悉多线程编程的人有利
* 除了CPU上下文开销外，其他的多线程存在的问题绿色线程也有

#### 回调风格
* 跟多线程程序完全不一样
* 线程／协程对程序员不可见
* 回调函数吞下了异常
* 回调函数无法把结果收集起来
* 回调连着回调不方便debug

### 更好的办法
在python3.3之前， 这就是所能做到的最好的结果了。要想更进一步改善，需要语言层面的支持，需要Python支持部分执行一个函数，在某个地方暂停执行然后保存下当前栈的所有对象和异常。有人会纳闷这不就是生成器么，协程允许一个函数返回一个列表，但一次只返回一个元素，然后暂停执行直到需要取出下一个元素时。但生成器的问题是一个生成器不能再调用一个生成器，然后让两个生成器都暂停，这个问题直到增加了yield from语法才被解决。生成器能维护栈的信息以及能抛出异常，再加上一个运行生成器的事件循环，就能构造出一个很厉害的异步的库了，asyncio就是这样一个库，你只需要在你的生成器前面加上一个@coroutine装饰器，asyncio库自动为你将这个生成器打补丁成一个协程。下面是一个像上面那样同样访问三个URL的例子：

```
import asyncio
import aiohttp

urls = ['http://www.google.com', 'http://www.yandex.ru', 'http://www.python.org']

@asyncio.coroutine
def call_url(url):
    print('Starting {}'.format(url))
    response = yield from aiohttp.ClientSession().get(url)
    data = yield from response.text()
    print('{}: {} bytes: {}'.format(url, len(data), data))
    return data

futures = [call_url(url) for url in urls]

asyncio.run(asyncio.wait(futures))
```

下面几点需要注意：
1. 没有检查错误的部分了，因为错误会被传递到合适的调用栈上
2. 可以返回对象了
3. 可以同时启动所有的协程，然后稍后再收集它们的结果
4. 没有回调函数
5. yield from没有返回之前下面一行是不会被执行的（是不是很像同步）

世界一下子变得美好了。唯一的问题就是yield from太像生成器了，如果真的使用生成器的话就会出问题了。

### async和await
asyncio受到了很大关注，Python官方甚至把它放到了标准库里。随着着加入了标准库，他们还在Python3.5增加了两个关键字async、await。这俩关键字更加明确的表明你的代码是异步的。async放在def 前边表明这个方法是异步的，await代替yield from更加明确的表明这是在等待一个协程的完成。下面是同样的例子但使用的async/await关键字。

```
import asyncio
import aiohttp

urls = ['http://www.google.com', 'http://www.yandex.ru', 'http://www.python.org']

async def call_url(url):
    print('Starting {}'.format(url))
    response = await aiohttp.ClientSession().get(url)
    data = await response.text()
    print('{}: {} bytes: {}'.format(url, len(data), data))
    return data

futures = [call_url(url) for url in urls]

asyncio.run(asyncio.wait(futures))
```

上面的代码大致的意思是这是一个异步的方法，当运行的时候，会返回一个可以被等待的协程。

### 抵达终点

![屏幕快照 2019-03-09 下午10.05.54.png](https://i.loli.net/2019/03/09/5c83c840c6201.png)

Python终于拥有了一个优秀的异步框架：asyncio。接下来我们看看它有没有解决多线程带来的那些问题。

* CPU上下文切换：asyncio是异步的并且使用了一个事件循环，它允许你的应用程序在等待IO时控制上下文切换，没有CPU的上下文切换。
* 竞态条件：asyncio在任一时刻只会运行一个协程，并且在你代码定义的地方切换协程，不会有竞态条件的问题
* 死锁／活锁：因为不必担心竞态条件，也就根本不需要使用锁了，但也有一种可能会遇到死锁，就是两个协程互相要唤醒对方，这是需要自己去避免的
* 饿死（Resource Starvation）：所有的协程是运行在一个线程上的，不需要额外的socket或者内存，因此很少会耗尽资源。不过Asyncio确实有一个executor pool， 其实就是一个线程池，如果在线程池里运行太多东西，可能会耗尽资源。然而使用executor pool本来就是个反常的模式，一般不会去这么做的

尽管asyncio已经非常棒了，它还是有一些问题的。首先，它比较新，可能会有一些奇怪的边缘场景。其次，如果你要彻底的异步，意味着你所有的代码都得是异步的，注意是每一行代码。因为同步的函数可能会花费很多时间从而阻塞你的事件循环。此外asyncio的配套的第三方库也都还不那么成熟，有时可能没法找到适合你技术栈的异步的第三方库。

这就是这次python异步之旅的全部内容了。使用Python进行异步编程有如下几个方式：绿色线程、回调以及真正的协程。在众多的选择之中，最好的无疑是asyncio。如果你的项目可以使用Python3.5及以上版本，你真的应该使用已经加入到标准库中的asyncio了。强烈建议你在下个项目中试一下asyncio，而不是继续使用多线程了。

由于译者水平的问题，有些内容理解可能不到位，请结合原文进行理解。

原文：

[Asynchronous Python: Await the Future](https://hackernoon.com/asynchronous-python-45df84b82434)




