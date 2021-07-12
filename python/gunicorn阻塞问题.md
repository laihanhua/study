## 一次大批量写文件报错排查
gunicorn项目发现，有些游戏同时多批且数量很大的文件时，数据库中的数据已经生成完毕，但是生成文件时发现文件没有完全写入（数量缺少且文件结束符EOF并没有写入），导致生成文件失败。

### 问题定位
gunicorn项目是通过gunicorn+gevent+flask方式启动，由于项目有【重推】机制，错误写入的文件已经被覆盖了。只能侥幸看日志有没有发现什么问题，发现当时文件生成失败时，报了这行错误：

```
run.log:
2021-03-12 09:57:45,146 $20431
	Traceback (most recent call last):
	  File "build/src/app/operate/gen_op.py", line 362, in generate
	  File "build/src/app/operate/gen_op.py", line 447, in write_sn_file
	  File "/usr/local/python-2.7.13/lib/python2.7/gzip.py", line 241, in write
	    self.fileobj.write(self.compress.compress(data))
	  File "/home/fee/runtime/mppython-2.7.13/lib/python2.7/site-packages/gunicorn/workers/base.py", line 192, in handle_abort
	    sys.exit(1)
	SystemExit: 1
```
从这个报错简单可以得到是在self.fileobj.write()写文件时候，worker异常退出了。当时有两个疑问：

1. self.fileobj.write()输入第三方库正常的写入文件，为什么会报错。

2. 有多个请求写入错误，但是报错日志只有一条。

带着疑问又看了一下gunicorn.err日志文件，在同一个时间点上也发现错误：

```
[2021-03-12 09:57:44 +0000] [20420] [CRITICAL] WORKER TIMEOUT (pid:20431)
[2021-03-12 09:57:45 +0000] [30925] [INFO] Booting worker with pid: 30925
```

日志表明是worker超时了，master kill掉旧的worker(20431)，创建了一个新的worker(30925)。那么梳理一下，整个案发的流程就是这样子的：

1. 项目是通过gunicorn+gevent方式启动，且配置4个worker。
2. 当请求批量调用cdkey_mana接口时，其中worker 20431承担了部分负载，worker创建greenlet协程处理请求。
3. 某个greenlet请求在处理self.fileobj.write()时，发生了异常，导致整个worker超时被kill掉。
4. 其他由此worker生成的greenlet协程也连带被kill调用，那么这一批任务写入文件都失败了，而报错只有1条。

### 写文件超时
要分析为什么写文件时会超时报错，从gunicorn工作原理和调度看起

#### gunicorn工作原理
1. gunicorn启动时，会生成一个master和多个worker（worker个数通过配置定义）

	```
	gunicorn/arbiter.py:
	  def run(self):
        "Main master loop."
        self.start()
        util._setproctitle("master [%s]" % self.proc_name)

        try:
            self.manage_workers()

            while True:
                self.maybe_promote_master()

                sig = self.SIG_QUEUE.pop(0) if len(self.SIG_QUEUE) else None
                if sig is None:
                    self.sleep()
                    self.murder_workers()
                    self.manage_workers()
                    continue
	
	```
	- self.start()会初始化出一个master进程，并读取配置文件等初始化操作
	- 通过manege_workers()，来fork出配置数量的worker
	- 在while True中，通过不断的self.murder\_workers/self.manage\_workers来管理和平衡worker
	
	```
	gunicorn/arbiter.py:
	  def murder_workers(self):
        """\
        Kill unused/idle workers
        """
        if not self.timeout:
            return
        workers = list(self.WORKERS.items())
        for (pid, worker) in workers:
            try:
            	   # mark
                if time.time() - worker.tmp.last_update() <= self.timeout: 
                    continue
            except (OSError, ValueError):
                continue

            if not worker.aborted:
                self.log.critical("WORKER TIMEOUT (pid:%s)", pid)
                worker.aborted = True
                self.kill_worker(pid, signal.SIGABRT)
            else:
                self.kill_worker(pid, signal.SIGKILL)
	```
	- 从mark那行代码可以看到，worker.tmp.last\_update()为worker上一次notify心跳的时间，如果worker超过了timeout还没有发送心跳，那么master就会kill掉这个worker，然后再通过manager\_worker来生成新的worker。
	
2. 生成gevent worker，每个worker都监听请求，并不断的notify()通知master

	```
	workers/ggevent.py：
	  def run(self):
        servers = []
        count = 0
        ssl_args = {}

        if self.cfg.is_ssl:
            ssl_args = dict(server_side=True, **self.cfg.ssl_options)

        for s in self.sockets:
			... 绑定server动作
        while self.alive:
            # mark lhh
            self.log.warning("lhh alive (pid:%s)" % self.pid)
            self.notify()
            gevent.sleep(1)
	```
	- 从while self.alive发现，worker会定时的调用self.notify()，notify会update某个tmp临时文件，文件的最新操作时间即为当前心跳上报时间。
	- master在上述worker.tmp.last_update()读取文件的修改时间来获取worker最新的活跃时间。
	
3. 每个gevent worker都有一个hub greenlet，用来控制每个请求的调度。当有新的请求进来时，worker会创建新的greenlet，hub切换到该greenlet逻辑处理。当有多个请求greenlet时，会执行流程切换。
	- 请求1进来，当前执行的协程从hub切换到greenlet1；hub --> greenlet1
	- 请求2进来，greenlet1执行某些异步io操作，此时切换会hub； greenlet1 --> hub 
	- hub接着切换到greenlet2；hub --> greenlet2

#### gevent阻塞模型

产生WORKER TIMEOUT超时，必然是worker没有snotify。通过上述gevent的调度可以发现当前worker没有切换会hub协程，在某一个greenlet中阻塞了，导致不能调用self.notify()。程序中的self.fileobj.write(self.compress.compress(data))是罪魁祸首。open/write为阻塞IO，greenlet执行此操作时，会阻塞切换，直到完成了write操作。那么就可能会导致worker不能够及时切换回hub，发送心跳。做如下测试：

```
1. worker设置为1个，timeout=10；并打印self.log.warning("lhh alive (pid:%s)" % self.pid)
2. gunicorn启动时，会不断打印上述日志。此时请求一次，在write文件时候io阻塞，不能切换回hub，10秒后worker超时，kill了旧worker并重新起了一个，继续上报心跳。部分日志日下：
[2021-03-22 15:22:53 +0800] [27120] [WARNING] lhh alive (pid:27120)
[2021-03-22 15:22:54 +0800] [27120] [WARNING] lhh alive (pid:27120)
[2021-03-22 15:23:04 +0800] [27115] [CRITICAL] WORKER TIMEOUT (pid:27120)
[2021-03-22 15:23:14 +0800] [27120] [INFO] Worker exiting (pid: 27120)
[2021-03-22 15:23:14 +0800] [27815] [INFO] Booting worker with pid: 27815
[2021-03-22 15:23:15 +0800] [27815] [WARNING] lhh alive (pid:27815)
[2021-03-22 15:23:16 +0800] [27815] [WARNING] lhh alive (pid:27815)
```

上述的测试已经能够定位并重现出问题，不过这里有个问题。对于gevent模拟，什么样的操作是阻塞的呢？最开始测试的时候，使用time.sleep/greet.sleep都不能够产生阻塞，为什么写文件write时就会发生？

后面google发现，在普通gevent模型的时候，time.sleep也是阻塞的，不过在gunicorn中会打猴子补丁monkey.patch_all()（也可以自行手动import补丁）。

猴子补丁会把time.sleep转化为greent.sleep非阻塞模型。除了time外，monkey可以为socket、dns、time、select、thread、os、ssl、httplib、subprocess、sys、aggressive、Event、builtins、signal模块打上的补丁，打上补丁后他们就是非阻塞的了。

```
workers/ggevent.py:
	class GeventWorker(AsyncWorker):

    server_class = None
    wsgi_handler = None

    def patch(self):
        from gevent import monkey
        monkey.noisy = False

        # if the new version is used make sure to patch subprocess
        if gevent.version_info[0] == 0:
            monkey.patch_all()
        else:
            monkey.patch_all(subprocess=True)
            
     def patch_all(socket=True, dns=True, time=True, select=True, thread=True, os=True, ssl=True, httplib=False,
              subprocess=True, sys=False, aggressive=True, Event=False,
              builtins=True, signal=True):
     
```

拿其中的patch_time()可以看到

```
def patch_time():
    """Replace :func:`time.sleep` with :func:`gevent.sleep`."""
    from gevent.hub import sleep
    import time
    patch_item(time, 'sleep', sleep)
```

**不过针对gevent文件IO，没有对其进行补丁，所以gevent里文件IO操作是不做切换的，即write写文件时会发生阻塞。所以我们在使用gunicorn写文件时，一定要注意！**
> 除了write之外，还发现os.popen也是阻塞的

针对这种阻塞模型，其实可以在代码中处理；例如我把patch_all()中的time部分注释掉，那么time.sleep也变成阻塞的了：

```
if time:
	#patch_time()
	pass
```

随后在程序中像捕捉异常一样，捕捉超时：

```
with gevent.Timeout(9,False) as timeout:
	# 9小于设置的timeout:
	time.sleep(10)
```
那么在9秒后，程序如果还没执行完time.sleep(10)，就会跑出异常并捕捉，程序继续往下走，不至于导致worker timeout（如果gevent.Timeout(11,False) 设置11>10，那么也会worker timeout）。

```
gevent/timeout.py:
class Timeout(BaseException):
    """
    Raise *exception* in the current greenlet after given time period::

        timeout = Timeout(seconds, exception)
        timeout.start()
        try:
            ...  # exception will be raised here, after *seconds* passed since start() call
        finally:
            timeout.cancel()
            
            ...
    To simplify starting and canceling timeouts, the ``with`` statement can be used::
    	
        with gevent.Timeout(seconds, exception) as timeout:
            pass  # ... code block ...            

```

### 解决方案
通过上述的分析结合我们的使用场景，最后决定通过设置大timeout=60，并且更细粒度读写文件，从每次写入50w行降低到10w行，减少单次write阻塞的时间，让worker在timeout时间内可以调度会hub并发心跳给master。

### 额外说明
1. 程序只有遇到io操作时（比如访问数据库），gevent才会进行调度。换句话说，在高IO密集型的程序下使用gevent模型很好。但是高CPU密集型的程序下没啥好处。
2. 如果程序真的时高CPU密集型，可以调用主动让出CPU的API之类（gevent.sleep），触发下一次调度。

### 参考文献
1. https://developer.aliyun.com/article/683284
2. https://www.cnblogs.com/xybaby/p/6370799.html
3. https://www.jianshu.com/p/861f29ac68e8
4. https://www.huaweicloud.com/articles/12455747.html
