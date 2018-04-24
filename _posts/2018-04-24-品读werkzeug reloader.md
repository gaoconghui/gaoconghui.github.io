---
layout: post
title: 品读 werkzeug reloader 实现机制
description: 细细的读读神级代码 werkzeug，研究其具体的实现逻辑，以及代码细节。
date: 2018.04.24 21:00 +08:00
tags: 
- python
- werkzeug
---

werkzeug使用reloader可以在文件被改变时自动加载更改过的文件，使用方法也很简单，`run_simple('localhost', 4000, application,use_reloader=True)`，ues_reloader=True即可。本文试图去品读一下reloader的实现以及一些小细节。

### 原理

先概述下整个reloader的原理，看起来会舒服一些。

非reloader的启动很简单，会调用`make_server`方法，然后调用`serve_forever()`去循环获取新的请求。

而reloader的机制，会起一个子进程，子进程有两个线程，一个线程会去跑server，一个线程去监控文件是否变动，如果文件发生变动，子进程会退出，并返回返回码3（自定义的返回码，标识因为文件变化而退出）。父进程检测子进程的退出码，并加以判断，如果是3，则重复上面的步骤，去再启动一次子进程，当然，此时加载的文件都会是新的文件了。

### 代码角度

接下来从代码的角度出发，看下整个流程。

#### 入口

```python
def inner():
    try:
        fd = int(os.environ['WERKZEUG_SERVER_FD'])
    except (LookupError, ValueError):
        fd = None
    srv = make_server(hostname, port, application, threaded,
                      processes, request_handler,
                      passthrough_errors, ssl_context,
                      fd=fd)
    if fd is None:
        log_startup(srv.socket)
    srv.serve_forever()

if use_reloader:
    # If we're not running already in the subprocess that is the
    # reloader we want to open up a socket early to make sure the
    # port is actually available.
    if os.environ.get('WERKZEUG_RUN_MAIN') != 'true':
        if port == 0 and not can_open_by_fd:
            raise ValueError('Cannot bind to a random port with enabled '
                             'reloader if the Python interpreter does '
                             'not support socket opening by fd.')

        # Create and destroy a socket so that any exceptions are
        # raised before we spawn a separate Python interpreter and
        # lose this ability.
        address_family = select_ip_version(hostname, port)
        s = socket.socket(address_family, socket.SOCK_STREAM)
        s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        s.bind(get_sockaddr(hostname, port, address_family))
        if hasattr(s, 'set_inheritable'):
            s.set_inheritable(True)

        # If we can open the socket by file descriptor, then we can just
        # reuse this one and our socket will survive the restarts.
        if can_open_by_fd:
            os.environ['WERKZEUG_SERVER_FD'] = str(s.fileno())
            s.listen(LISTEN_QUEUE)
            log_startup(s)
        else:
            s.close()

    # Do not use relative imports, otherwise "python -m werkzeug.serving"
    # breaks.
    from werkzeug._reloader import run_with_reloader
    run_with_reloader(inner, extra_files, reloader_interval,
                      reloader_type)
else:
    inner()
```

上面就是`use_reloader`起作用部分的代码。可以看到，使用了`use_reloader`之后相比较没加做了很多事情（废话 = = ）。接下去会挑这几行代码里的需要注意的点讲下。

* WERKZEUG_RUN_MAIN

  `WERKZEUG_RUN_MAIN`在这里其实还没赋值，看不太出具体的作用，可以在后面再看。初始肯定是null，**第一次**执行这几行代码的时候是会进入到if语句的（实际上这几行代码在每次代码更新执行reloader的时候都会重复进入，后面再说）

* can_open_by_fd

  这个参数是前面定义的，`can_open_by_fd = not WIN and hasattr(socket, 'fromfd')`，先不管windows系统下的情况，后面的fromdfd方法的解释如下

  >create a socket object from an open file descriptor [*]

  即从文件描述符创建一个socket。后面会创建一个socket，并把socket的文件描述符保存起来，方面传递。（实际上会在父进程子进程之间进行传递）

* socket.SO_REUSEADDR

  允许使用`TIME_WAIT`的端口。我们知道，TIME_WAIT状态下的端口是无法使用的，加上socket.SO_REUSEADDR参数后使这个socket的端口之后可以重复使用。

* 为什么直接创建一个socket，而不是在`inner`中使用`make_server`去创建？

  因为需要传递fd，在整个程序的入口需要先行创建。在后边我们会看到，子进程回去使用fd去创建socket（或者说是从fd恢复socket）

* inner

  在`use_reloader`为true的情况下，fd是存在的，会运行一个server，并且使用该fd对应的socket

在处理完是否为`WERKZEUG_RUN_MAIN`的情况后，程序进入`run_with_reloader`方法。

#### run_with_reloader

```python
def run_with_reloader(main_func, extra_files=None, interval=1,
                      reloader_type='auto'):
    """Run the given function in an independent python interpreter."""
    import signal
    reloader = reloader_loops[reloader_type](extra_files, interval)
    signal.signal(signal.SIGTERM, lambda *args: sys.exit(0))
    try:
        if os.environ.get('WERKZEUG_RUN_MAIN') == 'true':
            t = threading.Thread(target=main_func, args=())
            t.setDaemon(True)
            t.start()
            reloader.run()
        else:
            sys.exit(reloader.restart_with_reloader())
    except KeyboardInterrupt:
        pass
```

先往下看下`if`语句。同样的，第一次进入，还没赋值`WERKZEUG_RUN_MAIN`，会进去`sys.exit(reloader.restart_with_reloader())`，会把`reloader.restart_with_reloader()`的返回值作为程序的退出码。

`reloader_loops`是一个监控文件变化的类，有两个实现，分别是`StatReloaderLoop`以及`WatchDogReloaderLoop`，二者区别在于监控文件变动的方法不同。

#### ReloaderLoop

```python
class ReloaderLoop(object):
    name = None
    _sleep = staticmethod(time.sleep)

    def __init__(self, extra_files=None, interval=1):
      	# 接受 extra_files 参数，除了监控.py的变化以外，还会监控 extra_files 列表中所有文件的变化
        self.extra_files = set(os.path.abspath(x)
                               for x in extra_files or ())
        self.interval = interval

    def run(self):
        pass

    def restart_with_reloader(self):
        while 1:
            _log('info', ' * Restarting with %s' % self.name)
            # 获取到启动脚本，如['/usr/bin/python','test.py']
            args = _get_args_for_reloading()
            # 把环境变量(包括前面的fd等)
            new_environ = os.environ.copy()
            new_environ['WERKZEUG_RUN_MAIN'] = 'true'

            exit_code = subprocess.call(args, env=new_environ,
                                        close_fds=False)
            if exit_code != 3:
                return exit_code

    def trigger_reload(self, filename):
        self.log_reload(filename)
        sys.exit(3)

    def log_reload(self, filename):
        filename = os.path.abspath(filename)
        _log('info', ' * Detected change in %r, reloading' % filename)
```

`trigger_reload`方法是供子类去调用的，子类监控到文件的变化时会去调用`trigger_reload`，并且使进程退出，退出码为3（3在这里表示这因为文件变化而退出）

可以看到，`ReloaderLoop`的`restart_with_reloader`方法会去启动一个**子进程**，并赋予所有的环境变量（包括fd），子进程会去带上`WERKZEUG_RUN_MAIN`参数重新去跑下前面的`run_simple`方法。并且会捕获子进程的退出码，如上面讲的，如果返回的是3的话，表示文件变化而倒是子进程退出，直接重启就好了，即继续循环，启动子进程；如果程序是因为其他原因退出的，则返回返回码。

#### 子进程

接下来，我们看看子进程会做些什么。截止到上面的分析，我们知道，子进程相比较原先的父进程，目前唯一的泣别就是环境变量中`WERKZEUG_RUN_MAIN`为true，而这个字段会在两个地方会用到，一是最开始的`if use_reloader:`判断中，有这个字段的则不会去创建socket（毕竟父进程已经创建完成且把fd放在了环境变量中），二是`run_with_reloader`方法中。让我们再看下`run_with_reloader`

```python
def run_with_reloader(main_func, extra_files=None, interval=1,
                      reloader_type='auto'):
    """Run the given function in an independent python interpreter."""
    import signal
    reloader = reloader_loops[reloader_type](extra_files, interval)
    signal.signal(signal.SIGTERM, lambda *args: sys.exit(0))
    try:
        if os.environ.get('WERKZEUG_RUN_MAIN') == 'true':
            t = threading.Thread(target=main_func, args=())
            t.setDaemon(True)
            t.start()
            reloader.run()
        else:
            sys.exit(reloader.restart_with_reloader())
    except KeyboardInterrupt:
        pass

```

子进程到达了这个方法，会启动一个线程，运行`main_func`方法，也就是最开始的`inner`方法，用来启动一个server，该线程会被设置为`deamon`线程，即守护线程。守护线程会在其他线程退出后自动退出。

另外，reloader会运行`run()`方法，作用是监控文件的变化，并调用`trigger_reload`方法，在文件发生变化时退出，并返回3返回码。

还有一点，` signal.signal(signal.SIGTERM, lambda *args: sys.exit(0))`，这句看起来很简单，就是捕获`signal.SIGTERM`信号，也就是捕获kill或者是`ctrl + c`，并且退出。不过这里我还是有点疑问，为什么需要这个呢？加了信号之后唯一的区别，本来子进程退出会返回一个负数，加上之后会返回0。0代表着命令的成功执行，难道就是为了让程序更加'美丽'？

#### 再看ReloaderLoop

到了这里，整个流程算是理通了，就是我们一开始的原理。但还有一个问题我们之前一直选择跳过，就是`ReloaderLoop`的具体实现。我们前面说到，他有两个实现，分别为`StatReloaderLoop`以及`WatchdogReloaderLoop`。

```python
reloader_loops = {
    'stat': StatReloaderLoop,
    'watchdog': WatchdogReloaderLoop,
}

try:
    __import__('watchdog.observers')
except ImportError:
    reloader_loops['auto'] = reloader_loops['stat']
else:
    reloader_loops['auto'] = reloader_loops['watchdog']
```

接下来，我们会细致得去看一下具体的实现。

#####  StatReloaderLoop

```python
class StatReloaderLoop(ReloaderLoop):
    name = 'stat'

    def run(self):
        mtimes = {}
        while 1:
            for filename in chain(_iter_module_files(),
                                  self.extra_files):
                try:
                    mtime = os.stat(filename).st_mtime
                except OSError:
                    continue

                old_time = mtimes.get(filename)
                if old_time is None:
                    mtimes[filename] = mtime
                    continue
                elif mtime > old_time:
                    self.trigger_reload(filename)
            self._sleep(self.interval)
```

`StatReloaderLoop`的实现很简单，就是挨个去看文件的上次修改时间来确认文件是否发生改变，需要注意的是，如果interval比较小而文件又比较多的情况下，这个方法会很耗资源（显而易见），剩下的没啥好说的...

##### WatchdogReloaderLoop

`WatchdogReloaderLoop`依赖了第三方的库`watchdog`，这是一个可以监控文件变化的库，跨平台，运维用的比较多。他允许自定义监控一系列文件的变化，并在变化时调用相应的handler。

```python
class WatchdogReloaderLoop(ReloaderLoop):

    def __init__(self, *args, **kwargs):
        ReloaderLoop.__init__(self, *args, **kwargs)
        from watchdog.observers import Observer
        from watchdog.events import FileSystemEventHandler
        self.observable_paths = set()

        # 根据发生变化的文件名，确定是否需要重启(如果变动了一个不重要的小文件就没必要重启了)
        def _check_modification(filename):
            if filename in self.extra_files:
                self.trigger_reload(filename)
            dirname = os.path.dirname(filename)
            if dirname.startswith(tuple(self.observable_paths)):
                if filename.endswith(('.pyc', '.pyo', '.py')):
                    self.trigger_reload(filename)

        # 定义一个处理器类，分别处理不懂改变时要做的事(都是调用_check_modification方法)
        class _CustomHandler(FileSystemEventHandler):
            def on_created(self, event): _check_modification(event.src_path)
            def on_modified(self, event):  ...         
            def on_moved(self, event): ...
            def on_deleted(self, event): ...
                        
        self.observer_class = Observer
        self.event_handler = _CustomHandler()
        self.should_reload = False

    def trigger_reload(self, filename):
        # 调用会发生在handler中，退出没什么卵用，所以覆写了这个方法，让run循环自动退出
        self.should_reload = True
        self.log_reload(filename)

    def run(self):
        watches = {}
        observer = self.observer_class()
        observer.start()

        try:
            while not self.should_reload:
                # 使用watchdog去检查文件是否发生变化，并使用handler去处理。
        finally:
           # observer是一个线程，让observer也正常退出
            observer.stop()
            observer.join()
		# 返回3，标识文件发生变化
        sys.exit(3)
```

这部分代码很长，我把不太重要的省略掉了。代码比较简单，注释都卸载里边了，简单的说就是使用watchdog的方式去调用处理文件变化的事件，并按正常流程退出。

### 小结

werkzeug的代码真的很神，很多可以看的地方，比如父进程通过环境变量给子进程传递信息，父进程创建socket并获取其fd，子进程通过fd去创建socket，即便在重启的过程中也不至于`connection refused`，再比如使用退出码让子进程给父进程传递信息，再比如清晰的逻辑，各个环节的划分，reloader具体实现类的抽象等，都很值得学习。

我在看这代码之前想了很久，如果我来做reloader机制会如何去做，反正我做能实现功能就不错了...希望自己的代码有一天能这么好看吧。

