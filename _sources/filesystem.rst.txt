文件系统
==========

简单的文件读写是通过 ``uv_fs_*`` 函数族和与之相关的 ``uv_fs_t``
结构体完成的.

.. note::

    libuv 提供的文件操作和 :doc:`socket operations <networking>` 并不相同.
    套接字操作使用了操作系统本身提供了非阻塞操作, 而文件操作内部使用了阻塞函数,
    但是 libuv 是在线程池中调用这些函数, 并在应用程序需要交互时通知在事件循环中注册的监视器.

所有的文件操作函数都有两种形式 - 同步 *synchronous* 和  *asynchronous*.

同步 *synchronous* 形式如果没有指定回调函数则会被自动调用( **阻塞的** ),
函数的返回值和 Unix 系统的函数调用返回值相同(调用成功通常返回 0, 若出现错误则返回
-1).

而异步 *asynchronous* 形式则会在传入回调函数时被调用, 并且返回 0.

读写文件
--------

文件描述符可以采用如下方式获得:

.. code-block:: c

    int uv_fs_open(uv_loop_t* loop, uv_fs_t* req, const char* path, int flags, int mode, uv_fs_cb cb)

参数 ``flags`` 与 ``mode`` 和标准的
`Unix flags <http://man7.org/linux/man-pages/man2/open.2.html>`_ 相同.
libuv 会小心地处理 Windows 环境下的相关标志位(flags)的转换,
所以编写跨平台程序时你不用担心不同平台上文件打开的标志位不同。

关闭文件描述符可以使用:

.. code-block:: c

    int uv_fs_close(uv_loop_t* loop, uv_fs_t* req, uv_file file, uv_fs_cb cb)


与文件系统相关的操作的回调函数具有如下签名:

.. code-block:: c

    void callback(uv_fs_t* req);

让我们来看看 ``cat`` 命令的一个简单实现吧: 我们首先注册一个在文件打开时的回调函数
(顾名思义, 该函数将在文件打开时被调用).

.. rubric:: uvcat/main.c - opening a file
.. literalinclude:: ../code/uvcat/main.c
    :linenos:
    :lines: 39-48
    :emphasize-lines: 2

``uv_fs_t`` 的 ``result`` 字段在执行 ``us_fs_open`` 时代表一个文件描述符,
如果文件成功被打开, 我们开始读取文件.

.. warning::

    必须调用 ``uv_fs_req_cleanup()`` 来释放 libuv 内部使用的内存空间.

.. rubric:: uvcat/main.c - read callback
.. literalinclude:: ../code/uvcat/main.c
    :linenos:
    :lines: 24-37
    :emphasize-lines: 6,9,12

在调用 read 时, 你应该传递一个初始化的缓冲区, 在 read 回调函数被触发(调用之前),
该缓冲区将会被填满数据.

在 read 的回调函数中 ``result`` 如果是 0, 则读取文件时遇到了文件尾(EOF),
-1 则代表出现了错误, 而正整数则是表示成功读取的字节数.

此处给你展示了编写异步程序的通用模式, ``uv_fs_close()`` 是异步调用的.
*通常如果任务是一次性的, 或者只在程序启动和关闭时被执行的话都可以采用同步方式执行,
因为我们期望提高 I/O 效率, 采用异步编程时程序也可以做一些基本的任务并处理多路 I/O.*.
对于单个任务而言性能差异可以忽略, 但是代码却能大大简化.

我们可以总结出真正的系统调用返回值一般是存放在 ``uv_fs_t.result``.

写入文件与上述过程类似, 使用 ``uv_fs_write`` 即可.
*write 的回调函数在写入完成时被调用.*.  在我们的程序中回调函数只是只是简单地发起了下一次读操作,
因此, 读写操作会通过回调函数连续进行下去.

.. rubric:: uvcat/main.c - write callback
.. literalinclude:: ../code/uvcat/main.c
    :linenos:
    :lines: 14-22
    :emphasize-lines: 7

.. note::

    错误值通常保存在 `errno
    <http://man7.org/linux/man-pages/man3/errno.3.html>`_ 并可以通过
    ``uv_fs_t.errorno`` 获取, 但是被转换成了标准的 ``UV_*`` 错误码.
    目前还没有方法直接从 ``errorno`` 解析得到错误消息的字符串表示.

.. warning::

    由于文件系统和磁盘通常为了提高性能吞吐率而配置了缓冲区,
    libuv 中一次 '成功' 的写操作可能不会被立刻提交到磁盘上, 你可以通过 ``uv_fs_fsync``
    来保证一致性.

我们再来看看 ``main`` 函数中设置的多米诺骨牌吧(原作者意指在 main
中设置回调函数后会触发整个程序开始执行):

.. rubric:: uvcat/main.c
.. literalinclude:: ../code/uvcat/main.c
    :linenos:
    :lines: 50-54
    :emphasize-lines: 2

文件系统相关操作(Filesystem operations)
---------------------------------------

所有的标准文件系统操作, 例如 ``unlink``, ``rmdir``, ``stat``
都支持异步操作, 并且各个函数的参数非常直观. 他们和 read/write/open 的调用模式一致,
返回值都存放在 ``uv_fs_t.result`` 域. 完整的列表如下:

.. rubric:: Filesystem operations
.. literalinclude:: ../libuv/include/uv.h
    :lines: 1523-1581

回调函数中应该调用 ``uv_fs_req_cleanup()`` 函数来释放 ``uv_fs_t``
参数占用的内存.

.. _buffers-and-streams:

缓冲区与流(Buffers and Streams)
-------------------------------

libuv 中基本的 I/O 工具是流(``uv_stream_t``). TCP 套接字, UDP 套接字, 文件, 管道,
和进程间通信都可以作为 ``流`` 的子类.

``流`` (Streams) 通过每个子类特定的函数来初始化, 然后可以通过如下函数进行操作:

.. code-block:: c

    int uv_read_start(uv_stream_t*, uv_alloc_cb alloc_cb, uv_read_cb read_cb);
    int uv_read_stop(uv_stream_t*);
    int uv_write(uv_write_t* req, uv_stream_t* handle,
                uv_buf_t bufs[], int bufcnt, uv_write_cb cb);

基于流的函数比上面介绍的文件系统相关的函数更容易使用, libuv 在调用 ``uv_read_start``
后会自动从流中读取数据, 直到调用了 ``uv_read_stop``.

用于保存数据的单元被抽象成了 buffer 结构 -- ``uv_buf_t``.
它其实只保存了指向真实数据的指针(``uv_buf_t.base``) 以及真实数据的长度
(``uv_buf_t.len``). ``uv_buf_t`` 本身是轻量级的, 通常作为值被传递给函数,
真正需要进行内存管理的是 buffer 结构中的指针所指向的真实数据, 通常由应用程序申请分配并释放.

为了示范流的用法, 我们借助了(管道) ``uv_pipe_t`` ,
这使得我们把本地文件变成了流[#]_.  下面是利用 libuv 实现的一个简单的 ``tee`` .
将所有的操作变成了异步方式后, 事件 I/O 的强大能力便展现出来.
两个写操作并不会阻塞对方, 但是我们必须小心地拷贝数据至缓冲区,
并确保在写入数据之前缓冲区不被释放.

该程序按照如下方式执行::

    ./uvtee <output_file>

我们在指定的文件上打开了一个管道, libuv 的文件管道默认是双向打开的.

.. rubric:: uvtee/main.c - read on pipes
.. literalinclude:: ../code/uvtee/main.c
    :linenos:
    :lines: 62-81
    :emphasize-lines: 4,5,15

若是 IPC 或命名管道, ``uv_pipe_init()`` 的第三个参数应该设置为 1, 我们会在
:doc:`processes` 一节对此作出详细解释. 调用 ``uv_pipe_open()``
将文件描述符和文件关联在了一起.

我们开始监控标准输入 ``stdin``. 回调函数 ``alloc_buffer``
为程序开辟了一个新的缓冲区来容纳新到来的数据. ``read_stdin`` 也会被调用,
并且 ``uv_buf_t`` 作为调用参数.

.. rubric:: uvtee/main.c - reading buffers
.. literalinclude:: ../code/uvtee/main.c
    :linenos:
    :lines: 19-22,44-60

此处使用标准的 ``malloc`` 已经可以足够, 但是你也可以指定其他的内存分配策略.
例如, node.js 使用自己特定的 slab 分配器.

在任何情况下出错, read 回调函数 ``nread`` 参数都为 -1. 出错原因可能是
EOF(遇到文件尾), 在此种情况下我们使用 ''uv_close()'' 函数关闭所有的流,
``uv_close()`` 会根据所传递进来句柄的内部类型来自动处理. 如果没有出现错误,
``nread`` 是一个非负数, 意味着我们可以向输出流中写入 ``nread`` 字节的数据.
最后记住一点, 缓冲区 buffer 的分配和释放是由应用程序负责的,
所以记得释放不再使用的内存空间.

.. rubric:: uvtee/main.c - Write to pipe
.. literalinclude:: ../code/uvtee/main.c
    :linenos:
    :lines: 9-13,23-42

``write_data()`` 将读取的数据拷贝一份至缓冲区 ``req->buf.base``, 同样地,
当 write 完成后回调函数被调用时, 该缓冲区也并不会被传递到回调函数中, 所以,
为了绕过这一缺点, 我们将写请求和缓冲区封装在 ``write_req_t`` 结构体中,
然后在回调函数中解封该结构体来获取相关参数.

.. WARNING::

    If your program is meant to be used with other programs it may knowingly or
    unknowingly be writing to a pipe. This makes it susceptible to `aborting on
    receiving a SIGPIPE`_. It is a good idea to insert::

        signal(SIGPIPE, SIG_IGN)

    in the initialization stages of your application.

.. _aborting on receiving a SIGPIPE: http://pod.tst.eu/http://cvs.schmorp.de/libev/ev.pod#The_special_problem_of_SIGPIPE

文件变更事件(File change events)
---------------------------------

现代操作系统都提供了 API 用来在单独的文件或文件夹上设置监视器,
当文件被修改时应用程序会得到通知, libuv 也封装了常用的文件变更通知程序库 [#fsnotify]_.
这是 libuv 中最不一致的部分了,
文件变更通知系统本身在不同的系统中实现起来差别非常大,
因此让所有的事情在每个平台上都完美地工作将变得异常困难,
为了给出一个示例，我写了一个简单的工具, 该函数按照如下命令行运行，
并监视指定的文件.

    ./onchange <command> <file1> [file2] ...

文件变更通知通过 ``uv_fs_event_init()`` 启动:

.. rubric:: onchange/main.c - The setup
.. literalinclude:: ../code/onchange/main.c
    :linenos:
    :lines: 29-32
    :emphasize-lines: 3

第三个参数是实际监控的文件或者文件夹, 最后一个参数 ``flags`` 可取值如下:

.. literalinclude:: ../libuv/include/uv.h
    :lines: 1728,1737,1744

若设置 ``UV_FS_EVENT_WATCH_ENTRY`` 和 ``UV_FS_EVENT_STAT``
不做任何事情(目前).
设置了 ``UV_FS_EVENT_RECURSIVE`` 将会监视子文件夹(需 libuv 支持).

回调函数将接受以下参数:

  #. ``uv_fs_event_t *handle`` - 监视器. ``filename``
       字段是该监视器需要监视的文件.
  #. ``const char *filename`` - 如果监视目录, 则该参数指明该目录中发生了变更的文件,
       在 Linux 和 Windows 平台上可以是非 ``null``.
  #. ``int flags`` - ``UV_RENAME`` 或 ``UV_CHANGE``.
  #. ``int status`` - 目前为 0.

我们的例子只是简单地打印出参数, 并通过 ``system`` 函数运行指定命令.

.. rubric:: onchange/main.c - file change notification callback
.. literalinclude:: ../code/onchange/main.c
    :linenos:
    :lines: 9-18

----

.. [#fsnotify] inotify on Linux, FSEvents on Darwin, kqueue on BSDs,
               ReadDirectoryChangesW on Windows, event ports on Solaris, unsupported on Cygwin
.. [#] 参考 :ref:`pipes`
