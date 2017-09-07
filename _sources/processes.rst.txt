进程
====

libuv 也提供了进程管理的功能. libuv 进程管理抽象了不同平台的差异,
并通过流和管道使得进程间通信成为可能.

Unix 的设计哲学是:只做一件事，并把它做好(do one thing and do it well), 因此,
一个进程通常使用多个子进程来完成不同的任务, (类似于 shell 中的管道).
基于消息的多进程模型相对于共享内存式的多线程模型也更易于理解.

一个对事件驱动编程的通常指责是它不能利用现代计算机的多核优势.在一个多线程程序中，内核能通过调度和将不同
线程分配给不同的CPU核心来提高性能.但是一个事件循环只有一个线程，变通方法是启动多个进程，每个进程
运行一个事件循环，并将每个进程分配到单独的CPU核心中.


创建子进程(Spawning child processes)
------------------------------------

最简单的情形是你想要启动一个进程，并想要知道它什么时候结束.这可以通过 ``uv_spawn`` 完成.

.. rubric:: spawn/main.c
.. literalinclude:: ../code/spawn/main.c
    :linenos:
    :lines: 5-7,13-
    :emphasize-lines: 11,13-17

``uv_process_t`` 只是一个监视的结构,所有的选项通过 ``uv_process_options_t`` 来设置.简单的
启动一个进程,你只要设置 ``file`` 和 ``args`` 属性. ``file`` 是要执行的程序. ``uv_spwan`` 
在内部调用 execvp_ ,因此你不需要指定绝对路径.最后作为约定, **参数数组要比参数个数多一个,最后的
元素要被设置为NULL**.

.. _execvp: http://www.kernel.org/doc/man-pages/online/pages/man3/exec.3.html

在调用 ``uv_spwan`` 之后, ``uv_process_t.pid`` 保存了子进程的进程ID.

在进程即将结束或者收到导致结束的信号时,结束回调会被调用.

.. rubric:: spawn/main.c
.. literalinclude:: ../code/spawn/main.c
    :linenos:
    :lines: 9-12
    :emphasize-lines: 3

在进程结束之后 **需要** 关闭进程监视器.

改变进程参数(Changing process parameters)
-----------------------------------------

在子进程启动之前，可以通过 ``uv_process_potions_t`` 中的结构来设置执行环境.

改变执行目录(Change execution directory)
++++++++++++++++++++++++++++++++++++++++

设置 ``uv_process_options_t.cwd`` 参数改变进程的执行路径.

设置环境变量(Set environment variables)
+++++++++++++++++++++++++++++++++++++++

``uv_process_options_t.env`` 是一个字符串数组,每一个 ``VAR=VALUE`` 这样的结构用来
设置这个进程的环境变量.设置为 ``NULL`` 表示从父进程中继承环境变量.

选项参数(Option flags)
++++++++++++++++++++++

设置 ``uv_process_options_t.flags`` 下面的标志的按位或,可以修改子进程的行为.

* ``UV_PROCESS_SETUID`` - 设置子进程的用户ID为 ``uv_process_options_t.uid``.
* ``UV_PROCESS_SETGID`` - 设置子进程的用户组ID为 ``uv_process_options_t.gid``.

只有Unix支持修改UID/GID,在Windows上 ``uv_spawn`` 会失败，并返回 ``UV_ENOTSUP``

* ``UV_PROCESS_WINDOWS_VERBATIM_ARGUMENTS`` - 在Windows上对 ``uv_process_options_t.args`` 用引号和转义. Unix上忽略.
* ``UV_PROCESS_DETACHED`` - 在新会话中开始子进程, 子进程会在父进程退出后继续运行.具体看下面实例.

进程分离(Detaching processes)
-----------------------------

可以传人 ``UV_PROCESS_DETACHED`` 来启动守护进程.或者独立运行的子进程.这样父进程退出就不会影响
子进程.

.. rubric:: detach/main.c
.. literalinclude:: ../code/detach/main.c
    :linenos:
    :lines: 9-30
    :emphasize-lines: 12,19

要注意的是,监视器仍在监视子进程.所以你的程序不会退出.使用如果你想 *创建之后不再关心
(fire-and-forget)*，可以调用``uv_unref()``

向进程发送信号(Sending signals to processes)
--------------------------------------------

libuv在Unix封装了一个标准的 ``kill(2)`` 系统调用,并在Windows上提供了一个类似语义的实现.有 *一点警告* :
 ``SIGTERM`` ``SIGINT`` 和 ``SIGKILL`` , 都会导致进程结束. ``uv_kill`` 的函数签名是::

    uv_err_t uv_kill(int pid, int signum);

对于使用libuv启动的进程，你可以使用 ``uv_process_kill`` 来结束他.他的第一个参数是结构为 
``uv_process_t`` 的监视器,而不是进程的pid.此时 **记得调用** ``uv_close`` 来关闭监视器.

信号(Signals)
-------------

TODO: update based on https://github.com/joyent/libuv/issues/668

libuv提供一了Unix signals的封装，在Windows上也提供 `有限的支持
<https://github.com/joyent/libuv/blob/node-v0.9.4/include/uv.h#L1659>`_ .

为了信号在libuv上良好的运作，系统会把信号发送给 *所有时间循环上的所有处理器* .使用 ``uv_signal_init()``
来初始化一个处理器,并用一个事件循环关联它.在这个处理器上监听特定的信号.使用 ``uv_signal_start()`` 
注册处理函数.每个处理器只能关联一个信号.之后调用 ``uv_signal_start()`` 会覆盖之前的关联.
使用 ``uv_singnal_stop()`` 来结束监听.这里有个小例子来示范不同的情形.

.. rubric:: signal/main.c
.. literalinclude:: ../code/signal/main.c
    :linenos:
    :emphasize-lines: 8,17-18

发送 ``SIGUSR1`` 给这个进程，你会发现处理器调用了4次,每次都有一个 ``uv_signal_t`` .
这个处理器简单的停止了每个处理器，因此程序随之结束.发送个所有处理器的特性十分有用.一个拥有多个事件循环的服务程序能够保证在退出之前都安全的保存了数据,简单的为每个事件循环
添加一个 ``SIGINT`` 的监视器就可以.

子进程 I/O
----------

一个普通的新创建的进程有自己的一套文件描述符,0、1、2分表代表 ``stdin``、 ``stdout`` 和 ``stderr``.
有时候你想要和子进程共享文件描述符.比如:也许你的程序启动了一个子命令,想要将出错信息发送到log中去
,但是忽略 ``stdout`` .就是你想要将子进程的 ``stderr`` 显示出来.libuv支持 *继承* 文件描述
符.一个简单的例子,我们运行下面这个测试程序:

.. rubric:: proc-streams/test.c
.. literalinclude:: ../code/proc-streams/test.c

真正的程序 ``proc-stream`` 运行这个程序,并只继承 ``stderr``.子进程的文件描述符用
``uv_process_options_t`` 中的 ``stdio`` 来设置.首先设置 ``stdio_count`` 为想要设置的文件
描述符的数目. ``uv_process_options_t.stdio`` 是一个 ``uv_stdio_container_t`` 的数组,结构如下:

.. literalinclude:: ../libuv/include/uv.h
    :lines: 1263-1270

flags可以包含多个值.使用 ``UV_IGNORE`` 来制定不使用它.如果开始的三个 ``stdio`` 被设置为 ``UV_IGNORE``
它们会定向到 ``/dev/null`` 中.

既然我们想要传递已经获得的文件描述符，我们使用 ``UV_INHERIT_FD``.之后我们将 ``stderr`` 设置为 ``fd``.

.. rubric:: proc-streams/main.c
.. literalinclude:: ../code/proc-streams/main.c
    :linenos:
    :lines: 15-17,27-
    :emphasize-lines: 6,10,11,12

如果你运行 ``proc_stream`` ，你会发现只有"This is stderr"显示出来了.尝试将 ``stdout``
设置为继承的来观察输出.

将这中重定向适用到流上也很简单.通过设置 ``flags`` 为 ``UV_INHERIT_STREAM`` ,并设置 ``data.stream``
为父进程里面的流,子进程会将这些流当做标准IO.这可以用来实现 CGI_ 之类的东西.

.. _CGI: http://en.wikipedia.org/wiki/Common_Gateway_Interface

一个简单的CGI脚本/程序如下:

.. rubric:: cgi/tick.c
.. literalinclude:: ../code/cgi/tick.c

这个CGI服务程序结合了本章和 :doc:`networking` 里面的内容.每个客户端在练级关闭之前被发送了10次点滴.

.. rubric:: cgi/main.c
.. literalinclude:: ../code/cgi/main.c
    :linenos:
    :lines: 47,53-62
    :emphasize-lines: 5

这里我们简单的接受TCP连接,并把套接字(*流*)传递给 ``invoke_cgi_script`` .

.. rubric:: cgi/main.c
.. literalinclude:: ../code/cgi/main.c
    :linenos:
    :lines: 16, 25-45
    :emphasize-lines: 8-9,17-18

这个CGI脚本的 ``stdout`` 设置为套接字，所以每当我们的滴答脚本打印时，都会发送给客户端.通过使用
进程，我们将读/写的缓冲任务交给操作系统.从便利性上来讲，这十分不错.我们只需要担心创建进程是份耗时的操作.

.. _pipes:

管道(Pipes)
-----------

libuv的 ``uv_pipe_t`` 结构会让Unix程序员有些迷惑,因为它容易联系到 ``|`` 和 `pipe(7)`_ .
但是 ``uv_pipe_t`` 和匿名管道没有任何关系,何况他有两种用途:

#. Stream API - 它被当做 ``uv_stream_t`` API的具体实现来提供FIFO, 本地文件I/O的流接口.这在 :ref:`buffers-and-streams` 中用 ``uv_pipe_open`` 讨论过了.你也可以在TCP/UDP中使用它,不过它们已经有很方便的函数和结构.

#. IPC机制.``uv_pipe_t`` 可以使用 Unix域套接字(`Unix Domain Socket`_) 或者 Windows命名管道(`Windows Named Pipe`_) 实现来允许不同进程互相通讯.这点在后面讨论.

.. _pipe(7): http://www.kernel.org/doc/man-pages/online/pages/man7/pipe.7.html
.. _Unix Domain Socket: http://www.kernel.org/doc/man-pages/online/pages/man7/unix.7.html
.. _Windows Named Pipe: http://msdn.microsoft.com/en-us/library/windows/desktop/aa365590(v=vs.85).aspx

父子进程间通信
++++++++++++++

父进程和子进程可用通过管道来实现单向或者双向通讯. 这需要将 ``uv_stdio_container_t.flags`` 设置为
``UV_CREATE_PIPE`` 和 ``UV_READABLE_PIPE`` 或者 ``UV_WRITABLE_PIPE`` 的位组合.读/写标示是站在
子进程的角度观察的.

任意进程间通信
++++++++++++++

既然域套接字 [#]_ 可以很好的命名，并且在文件系统中有一个位置，那么可以用它来做无关进程之间的IPC.开源
桌面环境使用的 D-UBS_ 系统就使用域套接字作为事件通知.不同程序能过对一个上线联系或者新硬件检测做出
反应.MySQL数据库也运行一个域套接字，客户端可以通过它进行交互.

.. _D-BUS: http://www.freedesktop.org/wiki/Software/dbus

当使用域套接字时，套接字的创建/所有者充当服务器的角色，这就建立了一个客户端-服务端的关系.在初始化之后，通讯和TCP没有什么不同，所以我们重复使用我们的echo服务作为例子.

.. rubric:: pipe-echo-server/main.c
.. literalinclude:: ../code/pipe-echo-server/main.c
    :linenos:
    :lines: 56-
    :emphasize-lines: 5,9,13

我们给管道命名为 ``echo.sock`` ,这意味着它会在在本地目录上创建出来.在结合流API使用后，这个套接字和TPC套接字没有什么不同.
你可以使用 `netcat`_ 测试这个服务器::

    $ nc -U /path/to/echo.sock

一个客户端如果想要连接域套接字可以使用下面的函数::

    void uv_pipe_connect(uv_connect_t *req, uv_pipe_t *handle, const char *name, uv_connect_cb cb);

这里将 ``name`` 设置为 ``echo.sock`` 或者类似东西.

.. _netcat: http://netcat.sf.net

通过管道发送文件描述符(Sending file descriptors over pipes)
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

域套接字的一个很酷的特性是能够在进程之间传递文件描述符.这允许进程将他们的I/O交付给其他进程.
应用可以包含均衡负载进程，工作进程和其他方式来优化CPU的使用.

.. warning::

    在Windows上，只有表示TCP套接字的文件描述符能够被传递.

为了示范，我们来看一个采用流行的循环队列方式，将连接交付给工作进程的echo服务器.这个程序有点深入，并且只有片段
代码加入到本书中.推荐你阅读完整的代码来真正弄懂它.

工作进程十分简单，因为文件描述符是主进程传递过来的.

.. rubric:: multi-echo-server/worker.c
.. literalinclude:: ../code/multi-echo-server/worker.c
    :linenos:
    :lines: 7-9,53-
    :emphasize-lines: 7-9

``queue`` 是连接主进程的管道，通过它来传递文件描述符.我们使用 ``read2`` 函数来读取文件描述符. 重要的一点是，需要设置``uv_pipe_init`` 的 ``ipc`` 参数为1，来表示这个管道用作进程间通讯! 因为主进程将文件句柄写入工作进程的标准输入中，我们用 ``uv_pipe_open`` 来将管道连接到 ``stdin`` .

.. rubric:: multi-echo-server/worker.c
.. literalinclude:: ../code/multi-echo-server/worker.c
    :linenos:
    :lines: 36-52
    :emphasize-lines: 9

尽管 ``accpet`` 在代码中看起来有点奇怪，它实际是有意义的.``accept`` 传统意义上用来从另一个文件描述符(监听端)
获得一个文件描述符(客户端).这正是我们这里做的.从 ``queue``中将文件描述符( ``client`` )取回来.
从这点上来看，工作进程做着标准的echo服务的工作.

现在来看主进程.我们来看是如何启动工作进程来达到负载均衡.

.. rubric:: multi-echo-server/main.c
.. literalinclude:: ../code/multi-echo-server/main.c
    :linenos:
    :lines: 6-13

``child_worker`` 结构封装了子进程和他们各自连接主进程的管道.

.. rubric:: multi-echo-server/main.c
.. literalinclude:: ../code/multi-echo-server/main.c
    :linenos:
    :lines: 49,61-93
    :emphasize-lines: 15,18-19

在设置工作进程的过程中，我们使用灵活的liubv函数 ``uv_cpu_info`` 来得到CPU的个数.这样我们
能启动等数的工作进程.同样要注意，将第三个参数设置为1来将管道初始化为IPC通道.接下来我们将子进程
的 ``stdin`` 设置为一个可读的管道(从子进程的角度来看).到现在为止，一切都十分直观.工作进程们启动并等待文件描述符写入到他们的管道中.

我们在 ``on_new_connection`` (TCP是在``main()``函数中初始化的)函数中接受客户端套接字，并把它传递给循环队列里的下一个工作进程.

.. rubric:: multi-echo-server/main.c
.. literalinclude:: ../code/multi-echo-server/main.c
    :linenos:
    :lines: 29-47
    :emphasize-lines: 9,12-13

同样，``uv_write2`` 函数处理了所有的抽象，并且它就是简单的将文件描述符作为正确的参数传递过去. 至此我们的多进程echo服务就能够运转了.

TODO what do the write2/read2 functions do with the buffers?

----

.. [#] 在本章中域套接字也代表着Windows上的命名管道.
