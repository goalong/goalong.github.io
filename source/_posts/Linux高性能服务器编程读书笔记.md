---
title: Linux高性能服务器编程读书笔记
date: 2018-09-10
tags: 读书笔记
---

### I/O模型
socket在创建的时候默认是阻塞的，可以传递参数SOCK_NONBLOCK将其设为非阻塞的。阻塞I/O的系统调用可能因为无法立即完成而被操作系统挂起，直到等待的事情发生为止。socket的基础API中，可能被阻塞的系统调用包括accept,send,recv和connect.

非阻塞I/O执行的系统调用总是立即返回，而不管事件是否已经发生，如果事件没有立即发生，这些调用就返回-1。非阻塞I/O通常要和其他的I/O通知机制一起配合使用，比如I/O复用和SIGIO信号。

I/O复用是最常使用的I/O通知机制，它指的是，应用程序通过I/O复用函数向内核注册一组事件，内核通过I/O复用函数把其中就绪的事件通知给应用程序，linux上常用的I/O复用函数是select, poll和epoll_wait,I/O函数本身是阻塞的，它们能提高程序效率的原因在于它们具有同时监听多个I/O事件的能力。

SIGIO信号也可以用来报告I/O事件, 可以为一个目标文件描述符指定宿主进程，被指定的宿主进程可以捕获到SIGIO信号，这样，当目标文件描述符上有事件发生时，SIGIO信号的信号处理函数将被触发。

从理论上说， 阻塞I/O,I/O复用和信号驱动I/O都是同步I/O模型,因为在这三种io模型中，io的读写操作，都是在io事件发生之后，由应用程序来完成的。而POSIX规范所定义的异步io则不同. 对异步io而言，用户可以直接对io执行读写操作，这些操作告诉内核用户读写缓冲区的位置，以及io操作完成之后内核通知应用程序的方式，异步io的读写操作总是立即返回，而不论io是否是阻塞的，因为真正的读写操作已经由内核接管。也就是说，同步io模型要求用户代码自行执行io操作（将数据从内核缓冲区读入用户缓冲区，或将数据从用户缓冲区写入内核缓冲区),而异步io机制则是由内核来执行io操作（数据在内核缓冲区和用户缓冲区之间的移动由内核在后台完成), 可以这样认为，同步io向应用程序通知的是io就绪事件，异步io向应用程序通知的是io完成事件。

### I/O复用
 
#### select系统调用
用途是在一段指定时间内，监听用户感兴趣的文件描述符上的可读，可写和异常等事件。
 
 socket可读的情况：
 
 * socket内核接收缓冲区中的字节数大于等于其低水位标记SO_RCVLOWAT, 此时可以无阻塞的读该socket, 并且读操作返回的字节数大于0
 * socket通信的对方关闭连接，此时对socket的读操作将返回0
 * 监听socket上有新的连接请求
 * socket上有未处理的错误，此时可以使用getsocketopt来读取和清楚该错误

 socket可写的情况：
 
 * socket内核发送缓冲区的可用字节数大于等于其低水位标记，此时可以无阻塞的写该socket， 并且写操作返回的字节数大于0
 * socket的写操作被关闭，对写操作被关闭的socket执行写操作将触发一个SIGPIPE信号
 * socket使用非阻塞connect连接成功或者失败之后
 * socket上有未处理的错误，此时可以使用getsockopt来读取和清除该错误

#### poll
poll系统调用和select类似，也是在指定时间内轮询一定数量的文件描述符，以测试其中是否有酒须折。poll的原型如下：

	#include＜poll.h＞
	int poll(struct pollfd*fds,nfds_t nfds,int timeout);
	struct pollfd
	{
	  int fd;/*文件描述符*/
	  short events;/*注册的事件*/
	  short revents;/*实际发生的事件，由内核填充*/
	};
	
#### epoll
epoll是Linux特有的io复用函数，它在实现和使用上与select,poll有很大差异。首先，epoll使用一组函数来完成任务，而不是单个函数。其次，epoll把用户关心的文件描述符上的事件放在内核里的一个事件表中，从而无须像select和poll那样每次调用都要重复传入文件描述符集或事件集，但epoll需要使用一个额外的文件描述符，来唯一标识内核中的这个事件表，这个文件描述符使用epoll_create函数来创建：

	#include＜sys/epoll.h＞
	int epoll_create(int size)
size函数现在并不起作用，只是给内核一个提示，告诉它事件表需要多大。该函数返回的文件描述符用作其他所有epoll系统调用的第一个参数，以指定要访问的内核事件表。

下面的函数用来操作epoll的内核事件表：

	#include＜sys/epoll.h＞
	int epoll_ctl(int epfd,int op,int fd,struct epoll_event*event)
	
fd参数是要操作的文件描述符，op参数则制定操作类型，操作类型有如下三种：
* EPOLL_CTL_ADD，往事件表中注册fd上的事件
* EPOLL_CTL_MOD，修改fd上的注册事件
* EPOLL_CTL_DEL，删除fd上的注册事件

event参数指定事件，它是epoll_event结构指针类型，epoll_event的定义如下：

	struct epoll_event {
		__uint32_t events; // epoll事件
		epoll_data_t data; // 用户数据
	}
	
其中events成员描述事件类型。

epoll系列系统调用的主要接口是epoll_wait函数，它在一段超时时间内等待一组文件描述符上的事件，原型如下：

	#include＜sys/epoll.h＞
	// 该函数成功时返回就绪的文件描述符的个数，失败时返回-1并设置errno。
	int epoll_wait(int epfd,struct epoll_event*events,int maxevents,int timeout)
	
epoll_wait函数如果检测到事件，就将所有就绪的事件从内核事件表中复制到它的第二个参数events指向的数组中。这个数组只用于输出epoll_wait检测到的就绪事件，而不像select和poll的数组参数既用于传入用户注册的事件，又用于输出内核检测到的就绪事件，这就极大的提高了应用程序索引就绪文件描述符的效率。

#### LT和ET模式
epoll对文件描述符的操作有两种模式，LT(Level Trigger, 电平触发)模式和(Edge Trigger, 边沿触发)模式。LT模式是默认的工作模式，相当于一个效率较高的poll.当往epoll内核事件表中注册一个文件描述符上的EPOLLET事件时，epoll将以ET模式来操作该文件描述符，ET模式时俄 poll的高效工作模式。

对于采用LT工作模式的文件描述符，当epoll_wait检测到其上有事件发生并将此 通知应用程序后，应用程序可以不立即处理该事件。这样，当应用程序下一个调用epoll_wait时，epoll_wait还会再次向应用程序通告此事件，直到该事件被处理。而对于采用ET工作模式的文件描述符，当epoll检测到事件发生并通知应用程序后，应用程序必须立即处理该事件，因为后续的epoll_wait调用将不再向应用程序通知这一事件。可见，ET模式在很大程度上降低了同一个epoll事件被重复触发的次数，因此效率要比LT模式高。

我们期望的是一个socket连接在任一时刻都只被一个线程处理。这一点可以使用epoll的EPOLLONESHOT事件实现。

#### 三组IO复用函数的比较
select, poll和epoll三组IO复用系统调用都能同时监听多个文件描述符。它们将等待一定的超时时间，直到一个或多个文件描述符上有事件发生时返回，返回值时就绪的文件描述符的数量。返回0表示没有事件发生。

每次select和poll调用都返回整个用户注册的事件集合（其中包括就绪和未就绪的），所以应用程序索引就绪文件描述符的时间复杂度为O(n).epoll则在内核中维护一个事件表，并提供了一个系统调用epoll_ctl来控制网其中添加，修改，删除事件。这样，每次epoll_wait调用都能直接冲该内核事件表中取得用户注册的事件，而无须反复从用户空间读入这些事件。epoll_wait系统调用的events参数仅用来返回就绪的文件，使得应用程序索引就绪文件描述符的时间复杂度达到O(1).

poll和epoll_wait分别用nfds和maxevents参数指定监听多少个文件描述符合事件，这两个数值都能达到系统允许打开的最大文件描述符数目，即65535,而select允许监听的最大文件描述符数量通常有限制。

[![](https://ww3.sinaimg.cn/large/005YhI8igy1fv5q4qt8adj30p40cq0y9)](https://ww3.sinaimg.cn/large/005YhI8igy1fv5q4qt8adj30p40cq0y9)

### todo
第八章还需要好好理解并记下笔记。



 