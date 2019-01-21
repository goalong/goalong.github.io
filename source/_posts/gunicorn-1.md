---
title: Gunicorn源码阅读一：观其大略
date: 2018-05-28
tags: 
---

一直想了解一下gunicorn的源码实现，说干就干，把代码克隆到本地，打开pycharm, 一行行的运行来看。从网上搜了一堆的gunicorn源码阅读的文章，其中大部分都写的很好，自己其实根本没必要再去写了，但是考虑到
没有输出就很难以留下很深的印象，还是得自己动手来写一篇，最好是能在读完源码之后可以进行一个包含最简功能的原型的实现，这样才可以说理解了。

gunicorn采用pre-fork worker的服务器模型,pre-fork意味着一个主进程fork出许多子进程来处理请求。一个中心的主进程管理一系列的称为worker的子进程，主进程对客户端一无所知，所有的请求和响应都是完全由worker进程来处理。

主进程是一个简单的循环，监听一系列进程的信号并相应进行处理，它通过监听一些信号如TTIN, TTOU,  CHLD来管理一系列的运行中的worker。TTIN和TTOU告诉主进程增加或减少运行中worker的数量， CHLD表明子进程已经终结，因此主进程自动重启对应的worker进程。

最基本的以及默认的worker类型是同步worker，一次只能处理一个请求，同步worker不支持持久连接，即使手动加上Keep-Alive or Connection: keep-alive的头部，每次响应被发出之后也会将连接自动关闭。

### worker类型
	sync worekr, 同步worker，最基本的也是默认的worker类型，一次只能处理一个请求。
	async workers 是基于Greenlets(通过Evenlet和Gevent)的异步worker
	Tornado workers，不推荐
	AsyncIO workers， 使用python3的新特性的worker
	
一般使用(2 x $num_cores) + 1个worker

fork模型，来一个请求fork一个进程来处理
pre-fork, 预先创建大量进程，等待并处理接到的请求

### 配置本地调试环境
	git clone
	cd gunicorn
	virtualenv venv # 虚拟环境
	. venv/bin/activate
	pip install -r requirements_dev.txt 
	pip install gunicorn

安装完成之后可以启动一个测试中的例子：

	cd examples
	gunicorn test:app

应该就运行成功了，可以看到在test.py里有个函数叫app,是一个wsgi application.

通过setup.py里的gunicorn=gunicorn.app.wsgiapp:run可以看到，gunicorn命令实际调用的是gunicorn/app/wsgiapp.py里的run这个函数。这样一来，我们就可以用python -m gunicorn.app.wsgiapp -w 2 examples.test:app  的命令来启动，在IDE里配置一下，就可以方便的进行单步执行，断点等调试了。

### 大致流程
在IDE中一点点的单步执行可以看到gunicorn启动的大致流程，下面描述如下：

从wagiapp.py里可以看到，run函数调用了WSGIApplication("%(prog)s [OPTIONS] [APP_MODULE]").run()，也就是WSGIApplication这个对象的run方法，在执行run之前要先进行初始化，在BaseApplication的__init__方法进行了初始化，主要工作是从命令行或者配置文件里导入配置。当配置载入之后，进入到真正启动master进程的时候了：

    def run(self):
        try:
            Arbiter(self).run()
        except RuntimeError as e:
            print("\nError: %s\n" % e, file=sys.stderr)
            sys.stderr.flush()
            sys.exit(1)
大魔王Arbiter出现了，这个类似乎是解决我们所有疑惑的地方，前边的初始化导入配置这些都可以一掠而过，这里则必须要好好看了，这个类也就六百多行，所以可以全部认真看看。


在深入代码之前，可以进行一下测试，直观的了解一下gunicorn的工作流程，首先运行python -m gunicorn.app.wsgiapp -w 2 examples.test:app 这个命令来启动， 执行ps -ef|grep gunicorn  可以查到三个进程，一个主进程两个worker，然后运行kill -s $command $masterpid 这个形式的命令来向主进程发送信号， $masterpid是主进程的进程号， $command是信号的名字.

主进程接收的信号如下：

	QUIT 迅速关闭gunicorn
	TERM 仁慈的关闭，会等待worker执行完当前的请求再关闭
	HUP 重载配置，按新配置启动worker并且关闭老worker
	TTIN  增加worker
	TTOU 减少worker
	USR1 重新打开log文件
	USR2 在线升级gunicorn
	WINCH 如果主进程是守护进程则仁慈的关闭worker

worker进程可接收的信号：

	QUIT TERM USR1
	

可以使用kill -s $command $masterpid 配合 ps -ef|grep gunicorn 来观测各个信号的作用， 例如执行kill -s TTIN $masterpid 会发现新增了一个worker进程，kill -s TTOU $masterpid 则会减少一个worker进程。



参考：

[gunicorn官方文档](http://docs.gunicorn.org/en/stable/design.html)

[gunicorn代码阅读](http://rur.logdown.com/posts/2015/04/03/259224)

[gunicorn代码导读](http://gunicorn.readthedocs.io/en/latest/index.html)  
[gunicorn源码分析](https://blog.csdn.net/qq_33339479/article/details/78431209) 