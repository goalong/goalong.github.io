---
title: Gunicorn源码阅读三：Worker
date: 2018-06-07
tags: 
---

前面分析了Gunicorn主进程的实现，现在来看worker部分的实现，这里主要是看Sync Worker。

首先来看一下worker进程的创建：

    def spawn_worker(self):
        self.worker_age += 1
        worker = self.worker_class(self.worker_age, self.pid, self.LISTENERS,
                                   self.app, self.timeout / 2.0,
                                   self.cfg, self.log)
        self.cfg.pre_fork(self, worker)
        pid = os.fork()  # 这一句会分别在master进程和子进程中执行，子进程返回的pid=0
        if pid != 0: # 主进程执行，直接记录到self.WORKERS并返回
            worker.pid = pid
            self.WORKERS[pid] = worker
            return pid

        # Do not inherit the temporary files of other workers
        for sibling in self.WORKERS.values():
            sibling.tmp.close()

        # Process Child 子进程执行
        worker.pid = os.getpid()
        try:
            util._setproctitle("worker [%s]" % self.proc_name)
            self.log.info("Booting worker with pid: %s", worker.pid)
            self.cfg.post_fork(self, worker)
            worker.init_process()
            sys.exit(0)

master进程fork出子进程，可以看到在执行fork之前和之后会有pre_fork和post_fork调用，这算是Gunicorn提供的勾子，用来在fork之前和之后进行一些自定义的操作。fork之后， master进程将子进程的 pid记录到self.WORKERS然后就返回了。结下来的代码是子进程才会执行，重要的一句是worker.init_process(), 这一句开始了worker的真正工作，随后的代码都在workers目录下。

在看SyncWorker之前应该先看一下workers.base.Worker这个worker的基类，包括一下几个方法：

notify： 需要子类自己实现，用来向master进程每隔一段时间表明自己还活着，不然会被master进程给干掉

handle_xxx： 实际的信号处理方法

init_signals:  注册信号处理方法

run: 子类实现，

load_wsgi：导入wsgi application, 保存为属性self.wsgi

init_process: 在Arbiter中调用，可以算是worker的入口，会在这里调用init_signals和run方法

### init_process
下面重点看一下init_process方法：

	def init_process(self):
        """\
        If you override this method in a subclass, the last statement
        in the function should be to call this method with
        super(MyWorkerClass, self).init_process() so that the ``run()``
        loop is initiated.
        """

        # set environment' variables
        if self.cfg.env:
            for k, v in self.cfg.env.items():
                os.environ[k] = v

        util.set_owner_process(self.cfg.uid, self.cfg.gid,
                               initgroups=self.cfg.initgroups)

        # Reseed the random number generator
        util.seed()

        # For waking ourselves up
        self.PIPE = os.pipe()
        for p in self.PIPE:
            util.set_non_blocking(p)
            util.close_on_exec(p)

        # Prevent fd inheritance
        for s in self.sockets:
            util.close_on_exec(s)
        util.close_on_exec(self.tmp.fileno())
		 # 需要监测的文件描述符，socket以及管道
        self.wait_fds = self.sockets + [self.PIPE[0]]

        self.log.close_on_exec()

        self.init_signals() # 注册信号事件的处理方法

        # start the reloader
        if self.cfg.reload:
            def changed(fname):
                self.log.info("Worker reloading: %s modified", fname)
                self.alive = False
                self.cfg.worker_int(self)
                time.sleep(0.1)
                sys.exit(0)

            reloader_cls = reloader_engines[self.cfg.reload_engine]
            self.reloader = reloader_cls(extra_files=self.cfg.reload_extra_files,
                                         callback=changed)
            self.reloader.start()

        self.load_wsgi()  # 导入wsgi应用
        self.cfg.post_worker_init(self)

        # Enter main run loop
        self.booted = True
        self.run() 
        
init_process所作的主要工作有：

1. 如果有环境变量，设置环境变量
2. 注册信号处理方法
3. 导入wsgi应用
4. 调用run方法

然后来看run方法
### run

	def run(self):
        # if no timeout is given the worker will never wait and will
        # use the CPU for nothing. This minimal timeout prevent it.
        timeout = self.timeout or 0.5

        # self.socket appears to lose its blocking status after
        # we fork in the arbiter. Reset it here.
        for s in self.sockets:
            s.setblocking(0)

        if len(self.sockets) > 1:
            self.run_for_multiple(timeout)
        else:
            self.run_for_one(timeout)
   
基本上就是判断监听地址的数量，选择是调用run_for_one还是run_for_multiple.

无论是run_for_one还是run_for_multiple，本质上都是会有一个while循环不停调用self.accept, self.accept再调用self.handle, self.handle调用self.handle_request.

下面看一下handle_request

### handle_request

    def handle_request(self, listener, req, client, addr):
        environ = {}
        resp = None
        try:
            self.cfg.pre_request(self, req)
            request_start = datetime.now()
            resp, environ = wsgi.create(req, client, addr,
                    listener.getsockname(), self.cfg)
            # Force the connection closed until someone shows
            # a buffering proxy that supports Keep-Alive to
            # the backend.
            resp.force_close()
            self.nr += 1
            if self.nr >= self.max_requests:
                self.log.info("Autorestarting worker after current request.")
                self.alive = False
            respiter = self.wsgi(environ, resp.start_response)  # 这一句调用wsgi app来根据请求生成响应内容， 一般的web框架如Django, flask就是这时候被调用的
            try:
                if isinstance(respiter, environ['wsgi.file_wrapper']):
                    resp.write_file(respiter)
                else:
                    for item in respiter:
                        resp.write(item)
                resp.close()
                request_time = datetime.now() - request_start
                self.log.access(resp, req, environ, request_time)
                
resp, environ = wsgi.create(req, client, addr,
                listener.getsockname(), self.cfg) 这一句构造了Response对象以及环境变量environ, 然后调用self.wsgi(environ, resp.start_response) 来将请求交给wsgi application来处理, 常见的开发中使用web框架如Django, flask等，这里就是web框架开始接手的地方，wsgi application是一个可调用的对象（如函数，类等，有__call__方法), 接收environ和start_response两个参数。
                
下面是一个极简的wsgi application：

	def app(environ, start_response):
	    """Simplest possible application object"""
	
	    data = b'Hello, World!\n'
	    status = '200 OK'
	
	    response_headers = [
	        ('Content-type', 'text/plain'),
	        ('Content-Length', str(len(data))),
	        ('X-Gunicorn-Version', __version__),
	        #("Test", "test тест"),
	    ]
	    start_response(status, response_headers)
	    return iter([data])
	    
wsgi application通过environ这个字典里面的环境信息，例如请求的地址，方法，参数等，来构造响应的内容，然后调用start_response这个回调函数来将http状态码和头部返回给server，最后返回一个可迭代对象，内容是响应的正文。


最后总结一下worker的工作流程：

1. 主进程fork出worker进程，并调用init_process
2. init_process处理环境变量，导入wsgi app, 以及注册信号事件，最后调用run
3. run根据监听的socket的数量选择调用run_for_one或者run_for_multiple，而它们则都是调用accept来监听socket
4.  如果有请求到达，会交给 handle处理，handle调用handle_request, handle_request调用wsgi app来处理请求，wsgi app根据请求生成响应并返回给worker，然后worker将响应发送给客户端。



        
  




