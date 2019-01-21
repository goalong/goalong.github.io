---
title: Gunicorn源码阅读二：Arbiter大魔王
date: 2018-06-02
tags: 
---

Arbiter这个类是master进程工作的核心，主要的工作就是通过监听信号事件来维护worker进程的数量，以及通过SIGHUP/USR2等信号对应用进行热更新或者在线升级。

Arbiter类的外部调用只有Arbiter(self).run()这一句，也就是__init__和run两个方法，这里的self是一个WSGIApplication的实例。

Arbiter比较重要的方法有下列几个：

1. setup, start, __init__: 初始化以及导入app配置的几个方法，例如worker类型和数量，监听地址等
init_signals, signal: 关于信号处理的方法，接收到某个信号后会自动触发相应的处理方法
2. handle_xxx: 一系列以handle_作为前缀的方法，是各个信号对应的实际的处理方法
3. kill_worker, kill_workers: 杀死worker进程
4. spawn_worker, spawn_workers: 创建worker进程
5. run: 主循环, Arbiter的核心，通过不断循环来监听信号
6. sleep, wakeup: 休眠以及唤醒主循环

用一句话来概括Arbiter的工作其实就是，有一个主循环不断监听不同的信号事件，有信号的话调用相应的处理函数，以此来维护worker的数量。所以，主要就是这么个工作。下面来具体看一下几个方法：



### __init__, setup 方法
    def __init__(self, app):
        os.environ["SERVER_SOFTWARE"] = SERVER_SOFTWARE
        self._num_workers = None
        self._last_logged_active_worker_count = None
        self.log = None
        self.setup(app)
        self.pidfile = None
        self.systemd = False
        self.worker_age = 0
        self.reexec_pid = 0
        self.master_pid = 0
        self.master_name = "Master"

        cwd = util.getcwd()

        args = sys.argv[:]
        args.insert(0, sys.executable)

        # init start context 
        self.START_CTX = {
            "args": args,
            "cwd": cwd,
            0: sys.executable
        }
    def setup(self, app):
        self.app = app
        self.cfg = app.cfg

        if self.log is None:
            self.log = self.cfg.logger_class(app.cfg)

        # reopen files
        if 'GUNICORN_FD' in os.environ:
            self.log.reopen_files()

        self.worker_class = self.cfg.worker_class # worker类型
        self.address = self.cfg.address # 监听地址
        self.num_workers = self.cfg.workers # worker数量
        self.timeout = self.cfg.timeout
        self.proc_name = self.cfg.proc_name  # master进程名

        self.log.debug('Current configuration:\n{0}'.format(
            '\n'.join(
                '  {0}: {1}'.format(config, value.value)
                for config, value
                in sorted(self.cfg.settings.items(),
                          key=lambda setting: setting[1]))))

        # set enviroment' variables
        if self.cfg.env:
            for k, v in self.cfg.env.items():
                os.environ[k] = v

        if self.cfg.preload_app:  # 预先加载app
            self.app.wsgi()
        
可以看到主要是对一些属性赋值，进行初始化, setup方法获取到app的worker类型，配置worker数量，监听地址以及日志配置等

### run方法

    def run(self):
        "Main master loop."
        self.start()
        util._setproctitle("master [%s]" % self.proc_name)  # 设置master进程名

        try:
            self.manage_workers() # 调控worker数量

            while True:
                self.maybe_promote_master()

                sig = self.SIG_QUEUE.pop(0) if self.SIG_QUEUE else None  # 从信号队列里面取出最早的一个信号
                if sig is None:
                    self.sleep() # 进入睡眠或者立即
                    self.murder_workers()
                    self.manage_workers()
                    continue

                if sig not in self.SIG_NAMES:
                    self.log.info("Ignoring unknown signal: %s", sig)
                    continue

                signame = self.SIG_NAMES.get(sig)
                handler = getattr(self, "handle_%s" % signame, None)
                if not handler:
                    self.log.error("Unhandled signal: %s", signame)
                    continue
                self.log.info("Handling signal: %s", signame)
                handler()
                self.wakeup()
        except StopIteration:
            self.halt()
        except KeyboardInterrupt:
            self.halt()
        except HaltServer as inst:
            self.halt(reason=inst.reason, exit_status=inst.exit_status)
        except SystemExit:
            raise
        except Exception:
            self.log.info("Unhandled exception in main loop",
                          exc_info=True)
            self.stop(False)
            if self.pidfile is not None:
                self.pidfile.unlink()
            sys.exit(-1)
            
    def start(self):
        """\
        Initialize the arbiter. Start listening and set pidfile if needed.
        """
        self.log.info("Starting gunicorn %s", __version__)

        if 'GUNICORN_PID' in os.environ:
            self.master_pid = int(os.environ.get('GUNICORN_PID'))
            self.proc_name = self.proc_name + ".2"
            self.master_name = "Master.2"

        self.pid = os.getpid()
        if self.cfg.pidfile is not None:
            pidname = self.cfg.pidfile
            if self.master_pid != 0:
                pidname += ".2"
            self.pidfile = Pidfile(pidname)
            self.pidfile.create(self.pid)
        self.cfg.on_starting(self)

        self.init_signals()

        if not self.LISTENERS:
            fds = None
            listen_fds = systemd.listen_fds()
            if listen_fds:
                self.systemd = True
                fds = range(systemd.SD_LISTEN_FDS_START,
                            systemd.SD_LISTEN_FDS_START + listen_fds)

            elif self.master_pid:
                fds = []
                for fd in os.environ.pop('GUNICORN_FD').split(','):
                    fds.append(int(fd))

            self.LISTENERS = sock.create_sockets(self.cfg, self.log, fds)

        listeners_str = ",".join([str(l) for l in self.LISTENERS])
        self.log.debug("Arbiter booted")
        self.log.info("Listening at: %s (%s)", listeners_str, self.pid)
        self.log.info("Using worker: %s", self.cfg.worker_class_str)

        # check worker class requirements
        if hasattr(self.worker_class, "check_config"):
            self.worker_class.check_config(self.cfg, self.log)

        self.cfg.when_ready(self)
            
首先调用了start方法，start方法处理创建pid文件以及创建socket列表,并调用init_signals注册信号事件。master和worker之间通过信号来通信，

	def init_signals(self):
	        """\
	        Initialize master signal handling. Most of the signals
	        are queued. Child signals only wake up the master.
	        """
        # close old PIPE
        for p in self.PIPE:
            os.close(p)

        # initialize the pipe
        self.PIPE = pair = os.pipe()
        for p in pair:
            util.set_non_blocking(p)
            util.close_on_exec(p)

        self.log.close_on_exec()

        # initialize all signals
        for s in self.SIGNALS:
            signal.signal(s, self.signal)
        signal.signal(signal.SIGCHLD, self.handle_chld)

    def signal(self, sig, frame):
        if len(self.SIG_QUEUE) < 5:
            self.SIG_QUEUE.append(sig)
            self.wakeup()
    def wakeup(self):
        """\
        Wake up the arbiter by writing to the PIPE
        """
        try:
            os.write(self.PIPE[1], b'.')
        except IOError as e:
            if e.errno not in [errno.EAGAIN, errno.EINTR]:
                raise

可以看到除了SIGCHLD信号外，其他信号的handler都是signal方法，也就是接收到这些信号之后都会调用signal方法来处理，
signal方法的工作就是将信号加入队列，然后调用wakeup方法，向  self.PIPE 写入数据用于唤醒.

run方法在调用start方法之后调用了manage_workers方法，

	def manage_workers(self):
	        """\
	        Maintain the number of workers by spawning or killing
	        as required.
	        """
        if len(self.WORKERS.keys()) < self.num_workers:
            self.spawn_workers()

        workers = self.WORKERS.items()
        workers = sorted(workers, key=lambda w: w[1].age)
        while len(workers) > self.num_workers:
            (pid, _) = workers.pop(0)
            self.kill_worker(pid, signal.SIGTERM)

        active_worker_count = len(workers)
        if self._last_logged_active_worker_count != active_worker_count:
            self._last_logged_active_worker_count = active_worker_count
            self.log.debug("{0} workers".format(active_worker_count),
                           extra={"metric": "gunicorn.workers",
                                  "value": active_worker_count,
                                  "mtype": "gauge"})
                                  
                                  
self.WORKERS是保存当前运行的worker的字典，self.num_workers是想要维持的worker数量，通过对比当前运行的worker数量和num_workers, 少了就调用spawn_workers方法来创建，多了就调用kill_worker方法来杀死，这样就维持了num_workers数量的worker。


在调用manage_workers方法之后，就是一个while的无限循环，检查信号队列中有没有未处理的信号，如果没有信号，调用sleep方法，sleep方法检测管道是否可读，可读说明接收到信号了会调用signal方法往信号队列里添加, 继续调用manage_workers来维持worker数量；如果有信号则找到其handler, handler方法以handle_为前缀，信号名为后缀，方便直接通过信号名来找到，找到handler然后运行，然后继续等待。


### spawn_worker

	def spawn_worker(self):
        self.worker_age += 1
        worker = self.worker_class(self.worker_age, self.pid, self.LISTENERS,
                                   self.app, self.timeout / 2.0,
                                   self.cfg, self.log)
        self.cfg.pre_fork(self, worker)
        pid = os.fork()
        if pid != 0:
            worker.pid = pid
            self.WORKERS[pid] = worker
            return pid

        # Do not inherit the temporary files of other workers
        for sibling in self.WORKERS.values():
            sibling.tmp.close()

        # Process Child
        worker.pid = os.getpid()
        try:
            util._setproctitle("worker [%s]" % self.proc_name)
            self.log.info("Booting worker with pid: %s", worker.pid)
            self.cfg.post_fork(self, worker)
            worker.init_process()
            sys.exit(0)
        except SystemExit:
            raise
        except AppImportError as e:
            self.log.debug("Exception while loading the application",
                           exc_info=True)
            print("%s" % e, file=sys.stderr)
            sys.stderr.flush()
            sys.exit(self.APP_LOAD_ERROR)
        except:
            self.log.exception("Exception in worker process")
            if not worker.booted:
                sys.exit(self.WORKER_BOOT_ERROR)
            sys.exit(-1)
        finally:
            self.log.info("Worker exiting (pid: %s)", worker.pid)
            try:
                worker.tmp.close()
                self.cfg.worker_exit(self, worker)
            except:
                self.log.warning("Exception during worker exit:\n%s",
                                  traceback.format_exc())
                                  
 manage_workers中需要增加worker调用的是spawn_workers, 然后spawn_workers通过调用spawn_worker来创建worker。
 
 
 
 ### wakeup 和 sleep 方法
 
 主进程接收信号并处理的机制中，有几次出现了self.PIPE这个管道，我一直不太清楚具体是如何运作的，在sleep方法和wakeup方法里使用了self.PIPE，看到[gunicorn源码阅读笔记](https://www.zybuluo.com/orangleliu/note/302677)这篇文章里的方法，我也将run方法中的sleep那一行注释了，并且加了一行print(time.time())，然后运行，果然发现了一些端倪，while循环执行的非常快，使用top命令可以看到该进程CPU占用一直维持在90%以上，甚至95%以上，于是便明白了这是减少master进程的资源消耗
 
 def sleep(self):
        """\
        Sleep until PIPE is readable or we timeout.
        A readable PIPE means a signal occurred.
        """
        try:
            ready = select.select([self.PIPE[0]], [], [], 1.0) # 要么PIPE可读立即返回，要么等待超时
            if not ready[0]:
                return
            while os.read(self.PIPE[0], 1):
                pass
        except (select.error, OSError) as e:
            # TODO: select.error is a subclass of OSError since Python 3.3.
            error_number = getattr(e, 'errno', e.args[0])
            if error_number not in [errno.EAGAIN, errno.EINTR]:
                raise
        except KeyboardInterrupt:
            sys.exit()
            
    def wakeup(self):
        """\
        Wake up the arbiter by writing to the PIPE
        """
        try:
            os.write(self.PIPE[1], b'.')
        except IOError as e:
            if e.errno not in [errno.EAGAIN, errno.EINTR]:
                raise
注意sleep方法会调用select函数来检测管道是否可读，select函数的最后一个参数1.0时timeout的值，也就是select会花1秒钟来检测这个管道是否可读，超过1秒就返回，就是这个1秒钟的timeout，保证了没有信号时while循环不会循环的太快以至于CPU占用过高，在把sleep这个方法取消注释并运行后，发现进程CPU占用立即降到了0.1%左右，证实了我们的猜想。由此可以清楚的知道这个管道的作用了，当有信号到达时，往信号队列里面添加该信号，之后会调用wakeup方法，wakeup方法会往管道里写入数据，主循环在执行sleep的时候，select可以读取到管道中有数据而立即返回，不用再等待1秒的timeout。

sleep让master进程每一轮循环不要那么快，没信号的时候睡眠状态，减少资源消耗。

wakeup和sleep配合，master进程处睡眠的状态，当有信号时往管道里写数据，select检测到管道可读立即返回，不用继续休眠，随后在下一轮循环中从信号队列中取出信号进行处理。
                                  

最后来总结一下Arbiter的主要流程：

1. 根据传入app的配置进行初始化，例如worker的数量，worker的类型，注册信号处理函数等
2. 开始主循环，调用manage_workers维护一定数量的worker
3. 主循环从检测信号队列及管道，无信号则睡眠，有信号则立即唤醒并在下一个循环中处理信号



参考:

https://meta.tn/a/3e8eb2aa5bf4febad39b6138a873637b5b8f8696b0f92dc808df9a0357d5084e

http://rur.logdown.com/posts/2015/04/03/259224

https://blog.csdn.net/qq_33339479/article/details/78431209   

http://www.cnblogs.com/xybaby/p/6297089.html

