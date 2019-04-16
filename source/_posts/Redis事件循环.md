---
title: Redis事件循环
date: 2019-04-13
tags: redis
---
Redis服务器启动之后，会调用initServer进行初始化，并创建一个空的事件循环(EventLoop), 由一个事件循环的结构体保存事件循环的各种数据：

### aeEventLoop结构体

```c
/* 事件循环结构体 */
typedef struct aeEventLoop {
    int maxfd;   // 当前注册的最大描述符
    int setsize; // 需要监听的描述符个数
    long long timeEventNextId; // 下一个时间事件的id，用于生成时间事件的唯一标识
    time_t lastTime;     // 上一次事件循环的时间，用于检测系统时间是否变更
    aeFileEvent *events; // 注册要使用的文件事件
    aeFiredEvent *fired; // 已经就绪，待处理的事件
    aeTimeEvent *timeEventHead; // 时间事件头，时间事件相互连接组成了一个链表
    int stop; //  停止标识，1表示停止
    void *apidata; // 用于处理底层特定的API数据，对于epoll来说，其包括epoll_fd和epoll_event
    aeBeforeSleepProc *beforesleep; // 没有待处理事件时调用
} aeEventLoop;
```
### 创建处理新连接的文件事件
之后initServer会创建接收TCP或者UNIX域套接字的文件事件，在可读时调用acceptTcpHandler：

```c
 for (j = 0; j < server.ipfd_count; j++) {
        if (aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE,
            acceptTcpHandler,NULL) == AE_ERR)
            {
                serverPanic(
                    "Unrecoverable error creating server.ipfd file event.");
            }
    }
```
### acceptTcpHandler
acceptTcpHandler就是新的连接到来时的处理函数，下面来看这个函数的代码：

```c
void acceptTcpHandler(aeEventLoop *el, int fd, void *privdata, int mask) {
    int cport, cfd;
    char cip[REDIS_IP_STR_LEN];
    REDIS_NOTUSED(el);
    REDIS_NOTUSED(mask);
    REDIS_NOTUSED(privdata);
    // 接收客户端请求
    cfd = anetTcpAccept(server.neterr, fd, cip, sizeof(cip), &cport);
    if (cfd == AE_ERR) {
        redisLog(REDIS_WARNING,"Accepting client connection: %s", server.neterr);
        return;
  }
    redisLog(REDIS_VERBOSE,"Accepted %s:%d", cip, cport);
    acceptCommonHandler(cfd,0);
}
```
anetTcpAccept用来接收客户端请求，acceptCommonHandler会调用createClient, createClient非常重要，值得看一下代码：

```c
client *createClient(int fd) {
    client *c = zmalloc(sizeof(client));
    if (fd != -1) {
        anetNonBlock(NULL,fd);
        anetEnableTcpNoDelay(NULL,fd);
        if (server.tcpkeepalive)
            anetKeepAlive(NULL,fd,server.tcpkeepalive);
        if (aeCreateFileEvent(server.el,fd,AE_READABLE,
            readQueryFromClient, c) == AE_ERR)
        {
            close(fd);
            zfree(c);
            return NULL;
        }
    }
    ...
    return c
}
```
createClient主要做了两件事，一个是为新的连接创建了redisClient结构体来保存这个连接的信息，另一个是创建了一个文件事件，在文件可读的时候调用处理函数readQueryFromClient，readQueryFromClient顾名思义就是读取客户端发来的命令的，这下我们知道了Redis是如何读取客户端的请求的。
### 命令的执行
readQueryFromClient读取客户端发来的命令，随后调用processInputBuffer解析命令，processInputBuffer又调用processCommand，processCommand中会根据是单条命令还是多个来选择是直接执行还是放入队列：

```c

if (c->flags & CLIENT_MULTI &&
    c->cmd->proc != execCommand && c->cmd->proc != discardCommand &&
    c->cmd->proc != multiCommand && c->cmd->proc != watchCommand)
{
    queueMultiCommand(c); // 是多个命令则入队列
    addReply(c,shared.queued);
} else {
    call(c,CMD_CALL_FULL); // 单个命令调用call来执行
    c->woff = server.master_repl_offset;
    if (listLength(server.ready_keys))
        handleClientsBlockedOnLists();
}
```
如果是多个命令则放入队列，是单个命令则调用call来执行，call是核心的执行命令的函数:

```
oid call(redisClient *c, int flags) {
    ...
    // 执行命令
    c->cmd->proc(c);
    ...
}
```
c->cmd->proc(c)就是执行命令的代码。Redis初始化的时候创建了一个包含所有命令及其实现的数组，叫做redisCommand:

```
struct redisCommand redisCommandTable[] = {
    {"get",getCommand,2,"rF",0,NULL,1,1,1,0,0},
    {"set",setCommand,-3,"wm",0,NULL,1,1,1,0,0},
    {"setnx",setnxCommand,3,"wmF",0,NULL,1,1,1,0,0},
    {"setex",setexCommand,4,"wm",0,NULL,1,1,1,0,0},
    ...
}
```
redisCommand中包含了命令的字符串表示、具体实现函数以及默认参数等，
c->cmd->proc(c)就是通过在这个数组中查找到对应命令，然后执行其实现函数。

### 结果的返回
前面提到每个连接都创建了一个redisClient结构体对象，这个对象中的两个字段buffer和reply用来保存命令的返回值，buffer是一个固定大小的输出缓冲，reply是一个返回对象组成的链表，执行命令获得的结果会尝试放到这两个字端中，然后再返回给客户端。

在每个命令执行完毕之后都会调用到addReply一类的函数，以addReply为例：

```c
void addReply(client *c, robj *obj) {
    if (prepareClientToWrite(c) != C_OK) return;
    if (sdsEncodedObject(obj)) {
        if (_addReplyToBuffer(c,obj->ptr,sdslen(obj->ptr)) != C_OK)
            _addReplyObjectToList(c,obj);
    }
    ...
}...
```
addReply调用prepareClientToWrite, prepareClientToWrite根据client的flag判断是否将client放到server.clients_pending_write链表中。addReply随后尝试将结果放到输出缓冲中，如果超出缓冲的大小或者reply链表中已经有内容了，就将结果放到reply链表中。

随后在下一轮事件循环开始的时候，会执行eventLoop->beforesleep(eventLoop)这个函数，beforesleep会调用handleClientsWithPendingWrites函数：

```
int handleClientsWithPendingWrites(void) {
    ...
    //获取待输出结果的client数量
    int processed = listLength(server.clients_pending_write);
    ...
    while((ln = listNext(&li))) {
        ...
        //输出buf内容
        if (writeToClient(c->fd,c,0) == C_ERR) continue;
        ...
        //注册写事件处理器
        if (clientHasPendingReplies(c) &&
            aeCreateFileEvent(server.el, c->fd, AE_WRITABLE,
                sendReplyToClient, c) == AE_ERR)
        {
            freeClientAsync(c);
        }
    }
    return processed;
}
```

handleClientsWithPendingWrites遍历server.clients_pending_write，将每个client中保存的结果通过调用writeToClient发送给客户端。

如果writeToClient执行完之后输出缓冲和reply中还有内容，则会注册一个写事件，并关联处理函数sendReplyToClient，在后续的事件循环中会继续调用sendReplyToClient，sendReplyToClient内部调用了writeToClient继续向客户端发送数据。

### 事件循环

initServer执行完之后，main函数会调用aeMain进入一个包含while的事件循环(eventLoop)，这个事件循环可以说是Redis的核心了，循环会一直进行，直到事件循环的stop属性被设置为true时停止。

```c
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);
        aeProcessEvents(eventLoop, AE_ALL_EVENTS|AE_CALL_AFTER_SLEEP);
    }
}
```
### 事件处理器
可以看到aeMain中只是做了下判断是不是需要停止，不需要停止的话就一直调用aeProcessEvents来处理事件:

```c
/* 事件处理函数，返回处理的事件的个数 */
int aeProcessEvents(aeEventLoop *eventLoop, int flags)
{
    int processed = 0, numevents;

    // 没有需要处理的事件就直接返回
    if (!(flags & AE_TIME_EVENTS) && !(flags & AE_FILE_EVENTS)) return 0;

    // 即使没有要处理的文件事件，也需要调用select()函数
    // 这是为了睡眠直到下一个时间事件准备好。
    if (eventLoop->maxfd != -1 ||
        ((flags & AE_TIME_EVENTS) && !(flags & AE_DONT_WAIT))) {
        int j;
        aeTimeEvent *shortest = NULL;
        struct timeval tv, *tvp;
		// 获取最近的时间事件
        if (flags & AE_TIME_EVENTS && !(flags & AE_DONT_WAIT))
            shortest = aeSearchNearestTimer(eventLoop);
        if (shortest) {
            // 运行到这里说明时间事件存在，则根据最近可执行时间事件和现在的时间的时间差
            // 来决定文件事件的阻塞事件
            long now_sec, now_ms;

            aeGetTime(&now_sec, &now_ms);
            tvp = &tv;

            // 计算下一次时间事件准备好的时间
            long long ms =
                (shortest->when_sec - now_sec)*1000 +
                shortest->when_ms - now_ms;
            if (ms > 0) {
                tvp->tv_sec = ms/1000;
                tvp->tv_usec = (ms % 1000)*1000;
            } else {
                // 时间差小于0，代表可以处理了
                tvp->tv_sec = 0;
                tvp->tv_usec = 0;
            }
        } else {
            // 执行到这里，说明没有待处理的时间事件
            // 此时根据AE_DONT_WAIT参数来决定是否设置阻塞和阻塞的时间
            if (flags & AE_DONT_WAIT) {
                tv.tv_sec = tv.tv_usec = 0;
                tvp = &tv;
            } else {
                /* Otherwise we can block */
                tvp = NULL; /* wait forever */
            }
        }
		// 调用I/O复用函数获取就绪的事件
        numevents = aeApiPoll(eventLoop, tvp);
        for (j = 0; j < numevents; j++) {
            // 从已就绪事件中获取事件
            aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];
            int mask = eventLoop->fired[j].mask;
            int fd = eventLoop->fired[j].fd;
            int rfired = 0;

            // 是读事件则调用读事件处理函数
            if (fe->mask & mask & AE_READABLE) {
                rfired = 1;
                fe->rfileProc(eventLoop,fd,fe->clientData,mask);
            }
            // 是写事件则调用写事件处理函数
            if (fe->mask & mask & AE_WRITABLE) {
                if (!rfired || fe->wfileProc != fe->rfileProc)
                    fe->wfileProc(eventLoop,fd,fe->clientData,mask);
            }
            processed++;
        }
    }
    // 处理时间事件
    if (flags & AE_TIME_EVENTS)
        processed += processTimeEvents(eventLoop);
	// 返回处理的事件个数
    return processed;
}
```
aeProcessEvents主要的流程包括：

1. 计算等待时间，调用IO复用函数aeApiPoll等待相应的时间，来获取这段时间里就绪的文件事件
2. 对就绪的文件事件根据是可读还是可写分别调用其处理函数
3. 处理时间事件

注意到等待时间其实就是距离下一个时间事件的时间间隔。

参考：
redis设计与实现