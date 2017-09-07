线程
====

等等, 我们为什么会提到线程? 事件循环不应该本身就可以应对高并发网络编程么?
不一定, 线程仍然可以在 CPU 处理任务时来执行一些计算量相对较小的子任务,
即使在编程多线程程序中你必须编写大量同步原语, 但它们在大多时候还是可以派上大用场的.

线程在内部用来实现异步的系统调用. libuv 同样可以利用线程让你异步的完成一项会阻塞的任务. 办法是创建子线程,
并在子线程完成任务后获取其结果.

目前存在两个主流的线程库, Windows 线程库实现和 `pthreads`_. libuv 的线程库 API 与
pthread API 相似, 因此两者具有相同的语义.

值得注意的是, libuv 中的线程库是自包含的. 而其他特性都直接依赖事件循环或者回调.
事件循环和回调中的基本原则在线程中是完全不受限制的.
线程会在必要的时候被阻塞, 信号错误直接通过返回值来确定, 正如例子所示
:ref:`first example <thread-create-example>`, 线程中并不需要运行事件循环.

libuv 的线程 API 也非常有限, 因为不同平台上线程库的语义和语法也不同,
API 功能的完善程度也不尽相同.

本节做了如下假设: **一个线程里面(主线程)只有一个事件循环在运行**(**There is only one event loop,
running in one thread (the main thread)**). 没有其他的线程与事件循环交互
(除非使用 ``uv_async_send``). :doc:`multiple` 介绍了如何在多个线程中运行并管理事件循环.

线程核心操作(Core thread operations)
------------------------------------

下面的例子程序并没有太多代码, 只是通过 ``uv_thread_create`` 创建了线程, 然后调用
``uv_thread_join`` 等待线程退出.

.. _thread-create-example:

.. rubric:: thread-create/main.c
.. literalinclude:: ../code/thread-create/main.c
    :linenos:
    :lines: 26-37
    :emphasize-lines: 3-7

.. tip::

    在 Unix 平台上 ``uv_thread_t`` 只是 ``pthread_t`` 的别名,但是在实现细节上也避免依赖 pthread.

第二个参数是线程执行任务的入口函数地址, 最后 ``void *`` 类型的参数用来向线程传递自定义参数.
函数 ``hare`` 现在可以在另外一个单独线程中运行, 并由操作系统进行调度.

.. rubric:: thread-create/main.c
.. literalinclude:: ../code/thread-create/main.c
    :linenos:
    :lines: 6-14
    :emphasize-lines: 2

与 ``pthread_join()`` 允许目标线程通过 ``pthread_join()``
的第二个函数传回给调用线程一个返回值不同, ``uv_thread_join()``
并不允许调用/被调用线程之间通过这样的方式传值. 你应该使用 :ref:`inter-thread-communication`.

同步原语(Synchronization Primitives)
-------------------------------------

本节为了简洁起见并不完整简述线程的有关内容, 我在此也只归纳出一些 libuv 线程
API 比较特殊的地方,  剩下的你可以参看 pthread 的帮助页面 pthreads_.

互斥量(Mutexes)
~~~~~~~~~~~~~~~

libuv 中的互斥函数可以看作是 pthread 相关函数(pthread_mutex_*)的 **直接** 映射.

.. rubric:: libuv mutex functions
.. literalinclude:: ../libuv/include/uv.h
    :lines: 1834-1838

``uv_mutex_init()`` 和 ``uv_mutex_trylock()`` 在调用成功时返回 0,
若出错则返回 -1.

如果 `libuv` 以调试(debugging)模式进行编译, ``uv_mutex_destroy()``,
``uv_mutex_lock()`` 和 ``uv_mutex_unlock()`` 函数在出错时将会调用 ``abort()``.
同样地在调用 ``uv_mutex_trylock()`` 时, 除非错误码是 ``EAGAIN``,
否则也会导致程序中止(abort).

某些平台支持递归锁, 但是你不应该依赖某个特定平台的实现. BSD 互斥锁的实现规定:
如果在一个线程已经锁住了互斥量以后再次试图对该互斥量进行上锁, 则程序会抛出错误.
举个例子, 如下代码::

    uv_mutex_lock(a_mutex);
    uv_thread_create(thread_id, entry, (void *)a_mutex);
    uv_mutex_lock(a_mutex);
    // more things here

原本可以等待另外一个线程初始化后解锁, 但是在调试模式下该段代码会使你的程序崩溃,
或者在第二次调用 ``uv_mutex_lock`` 时出错.

.. note::

    Linux 平台的互斥锁支持递归, 但是 libuv 并没有暴露相关的 API.

锁(Locks)
~~~~~~~~~

读写锁是一种更细粒度的访问策略.两个线程可以同时访问共享内存区域,
当读线程拥有锁时, 写线程并不能获取到锁,
而当某个写线程拥有锁时其他读写线程都不能获得锁.
读写锁通常在数据库中用得比较多.以下是一个简单的例子:

.. rubric:: locks/main.c - simple rwlocks
.. literalinclude:: ../code/locks/main.c
    :linenos:
    :emphasize-lines: 13,16,27,31,42,55

运行该程序你会看到读线程有时候会相互重叠地执行, 但是在多个存在多个写线程时,
调度器会优先去调度写线程(写线程优先级更高), 因此, 如果你添加两个写线程,
你会看到只有当两个写线程都完成各自的任务后, 读线程才有机会获得锁.

其他
~~~~

libuv 也支持信号量 `semaphores`_, 条件变量 `condition variables`_ 和屏障 barriers_,
相关的 API 也和 pthread 中的相似.

在使用条件变量时, libuv 也可以在等待上设置超时时间, 不同平台超时设置可能会有所不同 [#]_.

.. _semaphores: http://en.wikipedia.org/wiki/Semaphore_(programming)
.. _condition variables: http://en.wikipedia.org/wiki/Condition_variable#Waiting_and_signaling
.. _barriers: http://en.wikipedia.org/wiki/Barrier_(computer_science)

另外, libuv 也提供了一个便于使用的函数``uv_once()`` (不要和 ``uv_run_once()`` 混淆).
多个线程以指定的函数指针为参数调用 ``uv_once`` 时,
**只有一个线程会成功调用, 并且仅被调用一次** ::

    /* Initialize guard */
    static uv_once_t once_only = UV_ONCE_INIT;

    int i = 0;

    void increment() {
        i++;
    }

    void thread1() {
        /* ... work */
        uv_once(once_only, increment);
    }

    void thread2() {
        /* ... work */
        uv_once(once_only, increment);
    }

    int main() {
        /* ... spawn threads */
    }

所有的线程都完成任务后, ``i == 1``.

.. _libuv-work-queue:

libuv 工作队列
--------------

``uv_queue_work()`` 是一个辅助函数, 它可以使得应用程序在单独的线程中运行某一任务,
并在任务完成后触发回调函数. ``uv_queue_work`` 看似简单, 但是在某些情况下却很实用,
因为该函数使得第三方库可以以事件循环的方式在你的程序中被使用.当你使用事件循环时,
应该 *确保在事件循环中运行的函数执行 I/O 任务时不被阻塞,
或者事件循环的回调函数不会占用太多 CPU 的计算能力*.
因为一旦发生了上述情况, 则意味着事件循环的执行速度会减慢, 事件得不到及时的处理.

但是也有一些代码在线程的事件循环的回调中使用了阻塞函数(例如执行 I/O 任务), 
(典型的 'one thread per client' 服务器模型), 并在单独的线程中运行某一任务.
libuv 只是提供了一层抽象而已.

下面是一个简单的例子(原型为 `node.js is cancer`_). 我们程序计算 fibonacci 数,
中途也会休眠一会,但是由于是在单独的线程中运行的, 所以阻塞和 CPU
计算密集型的任务(fibonacci 数的计算)并不会阻碍事件循环执行其它任务.

.. rubric:: queue-work/main.c - lazy fibonacci
.. literalinclude:: ../code/queue-work/main.c
    :linenos:
    :lines: 17-29

真正的执行任务的函数比较简单, 只是在单独的线程中执行. ``uv_work_t`` 结构是线索,
你可以通过 ``void* date`` 传递任意数据, 并且利用该指针和执行线程通信(数据传递).
但注意, 如果在多个线程中同时修改某个变量, 就应该使用合适的锁来保护共享变量.

*触发器* 是 ``uv_queue_work``:

.. rubric:: queue-work/main.c
.. literalinclude:: ../code/queue-work/main.c
    :linenos:
    :lines: 31-44
    :emphasize-lines: 40

线程函数会在单独的线程中被启动, 并传入 ``uv_work_t`` 结构, 一旦函数返回,
就会调用 ``after_fib`` 函数, 同时也传入 ``uv_work_t`` 结构的指针.

如果需要封装程序库的接口, 请参考 :ref:`pattern <baton>`
一节如何向线程传递数据.

.. _inter-thread-communication:

线程间通信(Inter-thread communication)
--------------------------------------

某些时候你想在同时运行的各个线程之间传递数据.
例如你可能在一个单独的线程中运行一个长时间任务(可能是通过 ``uv_queue_work``),
但是需要向主线程汇报任务进度. 一个简单的例子是下载管理器向用户汇报当前的下载任务的状态.

.. rubric:: progress/main.c
.. literalinclude:: ../code/progress/main.c
    :linenos:
    :lines: 7-8,34-
    :emphasize-lines: 2,11

异步线程通信是在事件循环上进行的, 因此尽管任何线程都可能成为消息发送者,
但是只有 libuv 事件循环的线程可以是接收者(或者事件循环本身是接收者). 
在任何时候异步监视器接受到了一个消息后, libuv 就会调用(``print_progress``)回调函数.

.. warning::

    应该注意: 消息的发送是异步的,回调函数应该在另外一个线程调用了
    ``uv_async_send`` 后立即被调用, 或者在稍后的某个时刻被调用.
    libuv 也有可能多次调用 ``uv_async_send`` 而只调用了一次回调函数.
    libuv 可以保证: 线程在调用了 ``uv_async_send`` 之后回调函数可至少被调用一次.
    如果你没有未调用的 ``uv_async_send``, 那么回调函数也不会被调用.
    如果你调用了两次(以上)的 ``uv_async_send``, 而 libuv
    暂时还没有机会运行回调函数, 则 libuv *可能* 会在多次调用 ``uv_async_send`` 后
    *只调用一次* 回调函数, 你的回调函数绝对不会在一次事件中被调用两次(或多次).

.. rubric:: progress/main.c
.. literalinclude:: ../code/progress/main.c
    :linenos:
    :lines: 10-23
    :emphasize-lines: 7-8

在下载函数中我们修改下载进度指示器, 并将消息写入 ``uv_async_send`` 发送队列,
记住: ``uv_async_send`` 也是非阻塞的, 它会立即返回.

.. rubric:: progress/main.c
.. literalinclude:: ../code/progress/main.c
    :linenos:
    :lines: 30-33

回调函数从监视器中提取数据, 这也是 libuv 的标准模式.

最后应记得清除监视器.

.. rubric:: progress/main.c
.. literalinclude:: ../code/progress/main.c
    :linenos:
    :lines: 25-28
    :emphasize-lines: 3

上面的例子滥用了 ``data`` 参数, bnoordhuis_ 指出使用 ``data`` 参数并不是线程安全的.
``uv_async_send()`` 实际上只会唤醒事件循环.
所以应该使用互斥量或者读写锁来保证对共享变量访问的顺序是正确的.

.. warning::

    互斥量和读写锁 **不能** 在信号处理函数中正常工作, 但是 ``uv_async_send`` 却可以.

``uv_async_send`` 的应用场景之一是不同功能的程序库之间需要线程级别的交互.
例如, node.js 的 v8 引擎实例, 上下文以及在 v8 示例启动时绑定的对象之间需要进行交互.
从另外一个线程直接访问 v8 执行引擎中的数据结构可能会导致程序不确定的行为.
考虑到 node.js 会绑定第三方库, 他们可能会如下工作:

1. 在 node 中, 创建第三方库的对象实例时会设置一个 JavaScript 回调函数用于获取更多信息::

    var lib = require('lib');
    lib.on_progress(function() {
        console.log("Progress");
    });

    lib.do();

    // do other stuff

2. ``lib.do`` 本来应该是非阻塞的, 但是第三方库却是阻塞的, 所以应该使用
   ``uv_queue_work``.

3. 在单独线程中完成的工作需要调用 progress 回调函数. 但是不能通过 JavaScript
   直接访问 v8 引擎, 以使用 ``uv_async_send`` 与 v8 进行交互.

4. 在主线程(v8 线程)设置的异步回调函数就可以通过 JavaScript 回调函数和 v8 引擎进行通行.

.. _pthreads: http://man7.org/linux/man-pages/man7/pthreads.7.html

----

.. _node.js is cancer: https://raw.github.com/teddziuba/teddziuba.github.com/master/_posts/2011-10-01-node-js-is-cancer.html
.. _bnoordhuis: https://github.com/bnoordhuis

.. [#] https://github.com/joyent/libuv/blob/master/include/uv.h#L1853
