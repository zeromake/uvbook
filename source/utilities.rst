工具集
======

本节归类了一些在编程中可能用到的工具和编程技术.
libev 的帮助页面 `page`_ 也介绍了一些有用的代码模式,
只需要简单地修改 API 的调用方式, libev 中的某些模式也可以应用到 libuv中.

定时器(Timers)
--------------

定时器在启动之后经过事先设置好的某一时间间隔就会调用回调函数.
libuv 的定时器也可以被设置为按照某一时间间隔重复调用回调函数, 而不是仅调用一次.

Libuv 定时器的使用非常简单, 如下代码所示, 初始化一个定时器, 然后启动它,
在你启动定时器的同时可以设置超时时间 ``timeout`` 和可选的 ``repeat`` 参数.

定时器可以在任何时候停止.

.. code-block:: c

    uv_timer_t timer_req;

    uv_timer_init(loop, &timer_req);
    uv_timer_start(&timer_req, callback, 5000, 2000);

将启动一个定期重复触发的定时器, 该定时器将会在执行了 ``uv_timer_start``
5 秒(``timeout``) 后执行回调函数, 然后每隔 2 秒(``repeat``)执行一次.调用

.. code-block:: c

    uv_timer_stop(&timer_req);

来停掉定时器. 该函数也可以在回调函数中调用.

定时器的重复时间间隔可以在任何时候通过函数设置::

    uv_timer_set_repeat(uv_timer_t *timer, int64_t repeat);

如果在回调函数中调用了该函数, 则意味着:

* 如果定时器是一次性的, 定时器已经停止, 则需要调用 ``uv_timer_start``.
* 如果定时器不是一次性的(not repeatedly), 并且下一次超时设置还未生效,
  那么旧的超时间隔还会被使用一次, 此后新的超时间隔才会生效.

辅助函数::

    int uv_timer_again(uv_timer_t *)

只对 **repeating 定时器(repeating timers)** 有效,
该函数和先停掉定时器然后再将 ``timeout`` 和 ``repeat``
参数设置为原始值并重启该定时器的效果一样.
如果定时器事先没有启动则该函数会出错(错误码 ``UV_EINVAL``)
并返回 -1.

下面是一个定时器的实际例子 :ref:`reference count section
<reference-count>`.

.. _reference-count:

事件循环引用计数(Event loop reference count)
--------------------------------------------

The event loop only runs as long as there are active watchers. This system
works by having every watcher increase the reference count of the event loop
when it is started and decreasing the reference count when stopped. It is also
possible to manually change the reference count of handles using::

    void uv_ref(uv_handle_t*);
    void uv_unref(uv_handle_t*);

These functions can be used to allow a loop to exit even when a watcher is
active or to use custom objects to keep the loop alive.

The former can be used with interval timers. You might have a garbage collector
which runs every X seconds, or your network service might send a heartbeat to
others periodically, but you don't want to have to stop them along all clean
exit paths or error scenarios. Or you want the program to exit when all your
other watchers are done. In that case just unref the timer immediately after
creation so that if it is the only watcher running then ``uv_run`` will still
exit.

The later is used in node.js where some libuv methods are being bubbled up to
the JS API. A ``uv_handle_t`` (the superclass of all watchers) is created per
JS object and can be ref/unrefed.

.. rubric:: ref-timer/main.c
.. literalinclude:: ../code/ref-timer/main.c
    :linenos:
    :lines: 5-8, 17-
    :emphasize-lines: 9

We initialize the garbage collector timer, then immediately ``unref`` it.
Observe how after 9 seconds, when the fake job is done, the program
automatically exits, even though the garbage collector is still running.

空闲监视器模式(Idle watcher pattern)
------------------------------------

The callbacks of idle watchers are only invoked when the event loop has no
other pending events. In such a situation they are invoked once every iteration
of the loop. The idle callback can be used to perform some very low priority
activity. For example, you could dispatch a summary of the daily application
performance to the developers for analysis during periods of idleness, or use
the application's CPU time to perform SETI calculations :) An idle watcher is
also useful in a GUI application. Say you are using an event loop for a file
download. If the TCP socket is still being established and no other events are
present your event loop will pause (**block**), which means your progress bar
will freeze and the user will think the application crashed. In such a case
queue up and idle watcher to keep the UI operational.

.. rubric:: idle-compute/main.c
.. literalinclude:: ../code/idle-compute/main.c
    :linenos:
    :lines: 5-9, 32-
    :emphasize-lines: 38

Here we initialize the idle watcher and queue it up along with the actual
events we are interested in. ``crunch_away`` will now be called repeatedly
until the user types something and presses Return. Then it will be interrupted
for a brief amount as the loop deals with the input data, after which it will
keep calling the idle callback again.

.. rubric:: idle-compute/main.c
.. literalinclude:: ../code/idle-compute/main.c
    :linenos:
    :lines: 10-19

.. _baton:

向工作者线程传递数据(Passing data to worker thread)
---------------------------------------------------

When using ``uv_queue_work`` you'll usually need to pass complex data through
to the worker thread. The solution is to use a ``struct`` and set
``uv_work_t.data`` to point to it. A slight variation is to have the
``uv_work_t`` itself as the first member of this struct (called a baton [#]_).
This allows cleaning up the work request and all the data in one free call.

.. code-block:: c
    :linenos:
    :emphasize-lines: 2

    struct ftp_baton {
        uv_work_t req;
        char *host;
        int port;
        char *username;
        char *password;
    }

.. code-block:: c
    :linenos:
    :emphasize-lines: 2

    ftp_baton *baton = (ftp_baton*) malloc(sizeof(ftp_baton));
    baton->req.data = (void*) baton;
    baton->host = strdup("my.webhost.com");
    baton->port = 21;
    // ...

    uv_queue_work(loop, &baton->req, ftp_session, ftp_cleanup);

Here we create the baton and queue the task.

Now the task function can extract the data it needs:

.. code-block:: c
    :linenos:
    :emphasize-lines: 2, 12

    void ftp_session(uv_work_t *req) {
        ftp_baton *baton = (ftp_baton*) req->data;

        fprintf(stderr, "Connecting to %s\n", baton->host);
    }

    void ftp_cleanup(uv_work_t *req) {
        ftp_baton *baton = (ftp_baton*) req->data;

        free(baton->host);
        // ...
        free(baton);
    }

We then free the baton which also frees the watcher.

轮询方式下的外部 I/O(External I/O with polling)
-----------------------------------------------

Usually third-party libraries will handle their own I/O, and keep track of
their sockets and other files internally. In this case it isn't possible to use
the standard stream I/O operations, but the library can still be integrated
into the libuv event loop. All that is required is that the library allow you
to access the underlying file descriptors and provide functions that process
tasks in small increments as decided by your application. Some libraries though
will not allow such access, providing only a standard blocking function which
will perform the entire I/O transaction and only then return. It is unwise to
use these in the event loop thread, use the :ref:`libuv-work-queue` instead. Of
course this will also mean losing granular control on the library.

The ``uv_poll`` section of libuv simply watches file descriptors using the
operating system notification mechanism. In some sense, all the I/O operations
that libuv implements itself are also backed by ``uv_poll`` like code. Whenever
the OS notices a change of state in file descriptors being polled, libuv will
invoke the associated callback.

Here we will walk through a simple download manager that will use libcurl_ to
download files. Rather than give all control to libcurl, we'll instead be
using the libuv event loop, and use the non-blocking, async multi_ interface to
progress with the download whenever libuv notifies of I/O readiness.

.. _libcurl: http://curl.haxx.se/libcurl/
.. _multi: http://curl.haxx.se/libcurl/c/libcurl-multi.html

.. rubric:: uvwget/main.c - The setup
.. literalinclude:: ../code/uvwget/main.c
    :linenos:
    :lines: 1-10,104-
    :emphasize-lines: 8,22,25-26

The way each library is integrated with libuv will vary. In the case of
libcurl, we can register two callbacks. The socket callback ``handle_socket``
is invoked whenever the state of a socket changes and we have to start polling
it. ``start_timeout`` is called by libcurl to notify us of the next timeout
interval, after which we should drive libcurl forward regardless of I/O status.
This is so that libcurl can handle errors or do whatever else is required to
get the download moving.

Our downloader is to be invoked as::

    $ ./uvwget [url1] [url2] ...

So we add each argument as an URL

.. rubric:: uvwget/main.c - Adding urls
.. literalinclude:: ../code/uvwget/main.c
    :linenos:
    :lines: 11-27
    :emphasize-lines: 15

We let libcurl directly write the data to a file, but much more is possible if
you so desire.

``start_timeout`` will be called immediately the first time by libcurl, so
things are set in motion. This simply starts a libuv `timer <Timers>`_ which
drives ``curl_multi_socket_action`` with ``CURL_SOCKET_TIMEOUT`` whenever it
times out. ``curl_multi_socket_action`` is what drives libcurl, and what we
call whenever sockets change state. But before we go into that, we need to poll
on sockets whenever ``handle_socket`` is called.

.. rubric:: uvwget/main.c - Setting up polling
.. literalinclude:: ../code/uvwget/main.c
    :linenos:
    :lines: 70-102
    :emphasize-lines: 9,11,16,19,23

We are interested in the socket fd ``s``, and the ``action``. For every socket
we create a ``uv_poll_t`` handle if it doesn't exist, and associate it with the
socket using ``curl_multi_assign``. This way ``socketp`` points to it whenever
the callback is invoked.

In the case that the download is done or fails, libcurl requests removal of the
poll. So we stop and free the poll handle.

Depending on what events libcurl wishes to watch for, we start polling with
``UV_READABLE`` or ``UV_WRITABLE``. Now libuv will invoke the poll callback
whenever the socket is ready for reading or writing. Calling ``uv_poll_start``
multiple times on the same handle is acceptable, it will just update the events
mask with the new value. ``curl_perform`` is the crux of this program.

.. rubric:: uvwget/main.c - Setting up polling
.. literalinclude:: ../code/uvwget/main.c
    :linenos:
    :lines: 29-57
    :emphasize-lines: 2,5-6,8,14,16

The first thing we do is to stop the timer, since there has been some progress
in the interval. Then depending on what event triggered the callback, we inform
libcurl of the same. Then we call ``curl_multi_socket_action`` with the socket
that progressed and the flags informing about what events happened. At this
point libcurl does all of its internal tasks in small increments, and will
attempt to return as fast as possible, which is exactly what an evented program
wants in its main thread. libcurl keeps queueing messages into its own queue
about transfer progress. In our case we are only interested in transfers that
are completed. So we extract these messages, and clean up handles whose
transfers are done.

检查并预备监视器(Check & Prepare watchers)
------------------------------------------

TODO

库的加载(Loading libraries)
---------------------------

libuv provides a cross platform API to dynamically load `shared libraries`_.
This can be used to implement your own plugin/extension/module system and is
used by node.js to implement ``require()`` support for bindings. The usage is
quite simple as long as your library exports the right symbols. Be careful with
sanity and security checks when loading third party code, otherwise your
program will behave unpredicatably. This example implements a very simple
plugin system which does nothing except print the name of the plugin.

Let us first look at the interface provided to plugin authors.

.. rubric:: plugin/plugin.h
.. literalinclude:: ../code/plugin/plugin.h
    :linenos:

.. rubric:: plugin/plugin.c
.. literalinclude:: ../code/plugin/plugin.c
    :linenos:

You can similarly add more functions that plugin authors can use to do useful
things in your application [#]_. A sample plugin using this API is:

.. rubric:: plugin/hello.c
.. literalinclude:: ../code/plugin/hello.c
    :linenos:

Our interface defines that all plugins should have an ``initialize`` function
which will be called by the application. This plugin is compiled as a shared
library and can be loaded by running our application::

    $ ./plugin libhello.dylib
    Loading libhello.dylib
    Registered plugin "Hello World!"

This is done by using ``uv_dlopen`` to first load the shared library
``libhello.dylib``. Then we get access to the ``initialize`` function using
``uv_dlsym`` and invoke it.

.. rubric:: plugin/main.c
.. literalinclude:: ../code/plugin/main.c
    :linenos:
    :lines: 7-
    :emphasize-lines: 14, 20, 25

``uv_dlopen`` expects a path to the shared library and sets the opaque
``uv_lib_t`` pointer. It returns 0 on success, -1 on error. Use ``uv_dlerror``
to get the error message.

``uv_dlsym`` stores a pointer to the symbol in the second argument in the third
argument. ``init_plugin_function`` is a function pointer to the sort of
function we are looking for in the application's plugins.

.. _shared libraries: http://en.wikipedia.org/wiki/Shared_library#Shared_libraries

TTY
---

Text terminals have supported basic formatting for a long time, with a `pretty
standardised`_ command set. This formatting is often used by programs to
improve the readability of terminal output. For example ``grep --colour``.
libuv provides the ``uv_tty_t`` abstraction (a stream) and related functions to
implement the ANSI escape codes across all platforms. By this I mean that libuv
converts ANSI codes to the Windows equivalent, and provides functions to get
terminal information.

.. _pretty standardised: http://en.wikipedia.org/wiki/ANSI_escape_sequences

The first thing to do is to initialize a ``uv_tty_t`` with the file descriptor
it reads/writes from. This is achieved with::

    int uv_tty_init(uv_loop_t*, uv_tty_t*, uv_file fd, int readable)

If ``readable`` is false, ``uv_write`` calls to this stream will be
**blocking**.

It is then best to use ``uv_tty_set_mode`` to set the mode to *normal* (0)
which enables most TTY formatting, flow-control and other settings. *raw* mode
(1) is also supported.

Remember to call ``uv_tty_reset_mode`` when your program exits to restore the
state of the terminal. Just good manners. Another set of good manners is to be
aware of redirection. If the user redirects the output of your command to
a file, control sequences should not be written as they impede readability and
``grep``. To check if the file descriptor is indeed a TTY, call
``uv_guess_handle`` with the file descriptor and compare the return value with
``UV_TTY``.

Here is a simple example which prints white text on a red background:

.. rubric:: tty/main.c
.. literalinclude:: ../code/tty/main.c
    :linenos:
    :emphasize-lines: 11-12,14,17,27

The final TTY helper is ``uv_tty_get_winsize()`` which is used to get the
width and height of the terminal and returns ``0`` on success. Here is a small
program which does some animation using the function and character position
escape codes.

.. rubric:: tty-gravity/main.c
.. literalinclude:: ../code/tty-gravity/main.c
    :linenos:
    :emphasize-lines: 19,25,38

The escape codes are:

======  =======================
Code    Meaning
======  =======================
*2* J    Clear part of the screen, 2 is entire screen
H        Moves cursor to certain position, default top-left
*n* B    Moves cursor down by n lines
*n* C    Moves cursor right by n columns
m        Obeys string of display settings, in this case green background (40+2), white text (30+7)
======  =======================

As you can see this is very useful to produce nicely formatted output, or even
console based arcade games if that tickles your fancy. For fancier control you
can try `ncurses`_.

.. _ncurses: http://www.gnu.org/software/ncurses/ncurses.html

----

.. [#] mfp is My Fancy Plugin
.. [#] I was first introduced to the term baton in this context, in Konstantin
       Käfer's excellent slides on writing node.js bindings --
       http://kkaefer.github.com/node-cpp-modules/#baton

.. _page: http://pod.tst.eu/http://cvs.schmorp.de/libev/ev.pod#COMMON_OR_USEFUL_IDIOMS_OR_BOTH
