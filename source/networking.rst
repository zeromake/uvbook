网络
====

libuv 的网络接口与 BSD 套接字接口存在很大的不同, 某些事情在 libuv 下变得更简单了,
并且所有接口都是都是非阻塞的, 但是原则上还是一致的.
另外 libuv 也提供了一些工具类的函数抽象了一些让人生厌的,
重复而底层的任务,比如使用 BSD 套接字结构来建立套接字, DNS 查询,
或者其他各种参数的设置.

libuv 中在网络 I/O 中使用了 ``uv_tcp_t`` 和 ``uv_udp_t`` 两个结构体.

TCP
---

TCP 是一种面向连接的流式协议, 因此是基于 libuv 的流式基础架构上的.

服务器(Server)
++++++++++++++

服务器端的 sockets 处理流程如下:

1. ``uv_tcp_init`` 初始化 TCP 监视器.
2. ``uv_tcp_bind`` 绑定.
3. 在指定的监视器上调用 ``uv_listen`` 来设置回调函数, 当有新的客户端连接到来时,
   libuv 就会调用设置的回调函数.
4. ``uv_accept`` 接受连接.
5. 使用 :ref:`stream operations <buffers-and-streams>` 与客户端进行通信.

以下是一个简单的 echo 服务器的例子:

.. rubric:: tcp-echo-server/main.c - The listen socket
.. literalinclude:: ../code/tcp-echo-server/main.c
    :linenos:
    :lines: 50-
    :emphasize-lines: 4-5,7-9

你可以看到辅助函数 ``uv_ip4_addr`` 用来将人为可读的字符串类型的 IP
地址和端口号转换成 BSD 套接字 API 所需要的 ``struct sockaddr_in`` 类型的结构.
逆变换可以使用 ``uv_ip4_name`` 来完成.

.. NOTE::

    对于 IPv6 来说应该使用 ``uv_ip6_*`` 形式的函数.

大部分的设置(setup)函数都是普通函数, 因为他们都是 ``计算密集型(CPU-bound)``,
直到调用了 ``uv_listen`` 我们才回到 libuv 中回调函数风格.
``uv_listen`` 的第二个参数 backlog 队列长度 -- 即连接队列最大长度.

当客户端发起了新的连接时, 回调函数需要为客户端套接字设置一个监视器,
并调用 ``uv_accept`` 函数将客户端套接字与新的监视器在关联一起.
在例子中我们将从流中读取数据.

.. rubric:: tcp-echo-server/main.c - Accepting the client
.. literalinclude:: ../code/tcp-echo-server/main.c
    :linenos:
    :lines: 34-48
    :emphasize-lines: 9-10

剩余部分的函数与上一节流式例子中的代码相似, 你可以在例子程序中找到具体代码,
如果套接字不再使用记得调用 ``uv_close`` 关闭该套接字.
如果你不再接受连接, 你可以在 ``uv_listen`` 的回调函数中关闭套接字.

客户端(Client)
++++++++++++++

在服务器端你需要调用 bind/listen/accept, 而在客户端你只需要调用 ``uv_tcp_connect``.
``uv_tcp_connect`` 使用了与 ``uv_listen`` 风格相似的回调函数 ``uv_connect_cb``
如下::

    uv_tcp_t socket;
    uv_tcp_init(loop, &socket);

    uv_connect_t connect;

    struct sockaddr_in dest = uv_ip4_addr("127.0.0.1", 80);

    uv_tcp_connect(&connect, &socket, dest, on_connect);

建立连接后会调用 ``on_connect``.

UDP
---

`User Datagram Protocol`_ 提供了无连接, 不可靠网络通信协议, 因此 libuv
并不提供流式 UDP 服务, 而是通过 ``uv_udp_t`` 结构体(用于接收)和
`uv_udp_send_t` 结构体(用于发送)以及相关的函数给开发人员提供了非阻塞的 UDP
服务. 所以, 真正读写 UDP 的函数与普通的流式读写非常相似.为了示范如何使用 UDP,
下面提供了一个简单的例子用来从 `DHCP`_ 获取 IP 地址. -- DHCP 发现.

.. note::

    你应该以 **root** 用户运行 ``udp-dhcp``, 因为该程序使用了端口号低于 1024 的端口.

.. rubric:: udp-dhcp/main.c - Setup and send UDP packets
.. literalinclude:: ../code/udp-dhcp/main.c
    :linenos:
    :lines: 7-10,104-
    :emphasize-lines: 8,10-11,14,15,21

.. note::

    ``0.0.0.0`` 地址可以绑定本机所有网口. ``255.255.255.255`` 是广播地址,
    意味着网络包可以发送给子网中所有网口, 端口 ``0`` 说明操作系统可以任意指定端口进行绑定.

首先我们在 68 号端口上设置了绑定本机所有网口的接收套接字(DHCP 客户端),
并且设置了读监视器. 然后我们利用相同的方法设置了一个用于发送消息的套接字.
并使用 ``uv_udp_send`` 在 67 号端口上(DHCP 服务器)发送 *广播消息*.

设置广播标志也是 **必要** 的, 不然你会得到 ``EACCES`` 错误 [#]_.
发送的具体消息与本书无关, 如果你对此感兴趣, 可以参考源码. 若出错,
则读写回调函数会收到 -1 状态码.

由于 UDP 套接字并不和特定的对等方保持连接,
所以 read 回调函数中将会收到用于标识发送者的额外信息. 如果缓冲区是由你自己的分配的,
并且不够容纳接收的数据, 则``flags`` 标志位可能是 ``UV_UDP_PARTIAL``.
*在这种情况下, 操作系统会丢弃不能容纳的数据.* (这也是 UDP 为你提供的特性).

.. rubric:: udp-dhcp/main.c - Reading packets
.. literalinclude:: ../code/udp-dhcp/main.c
    :linenos:
    :lines: 15-27,38-41
    :emphasize-lines: 1,16

UDP 选项(UDP Options)
+++++++++++++++++++++

生存时间TTL(Time-to-live)
~~~~~~~~~~~~~~~~~~~~~~~~~

可以通过 ``uv_udp_set_ttl`` 来设置网络数据包的生存时间(TTL).

仅使用 IPv6 协议
~~~~~~~~~~~~~~~~

IPv6 套接字可以同时在 IPv4 和 IPv6 协议下进行通信. 如果你只想使用 IPv6
套接字, 在调用 ``uv_udp_bind6`` [#]_ 时请传递 ``UV_UDP_IPV6ONLY`` 参数.

多播(Multicast)
~~~~~~~~~~~~~~~

套接字可以使用如下函数订阅(取消订阅)一个多播组:

.. literalinclude:: ../libuv/include/uv.h
    :lines: 796-798

``membership`` 取值可以是 ``UV_JOIN_GROUP`` 或 ``UV_LEAVE_GROUP``.

多播包的本地回路是默认开启的 [#]_, 可以使用 ``uv_udp_set_multicast_loop`` 来开启/关闭该特性.

多播包的生存时间可以使用 ``uv_udp_set_multicast_ttl`` 来设置.

DNS 查询(Querying DNS)
----------------------

libuv 提供了异步解析 DNS 的功能, 用于替代 ``getaddrinfo`` [#]_.
在回调函数中, 你可以在获得的 IP 地址上执行普通的套接字操作.
让我们通过一个简单的 DNS 解析的例子来看看怎么连接 ``Freenode`` 吧:

.. rubric:: dns/main.c
.. literalinclude:: ../code/dns/main.c
    :linenos:
    :lines: 61-
    :emphasize-lines: 12

如果 ``uv_getaddrinfo`` 返回非零, 表示在建立连接时出错, 你设置的回调函数不会被调用,
所有的参数将会在 ``uv_getaddrinfo`` 返回后被立即释放. 有关 `hostname`, `servname` 和  `hints`
结构体的文档可以在 getaddrinfo_ 帮助页面中找到.

在解析回调函数中, 你可以在 ``struct addrinfo(s)`` 结构的链表中任取一个 IP.
这个例子也演示了如何使用 ``uv_tcp_connect``. 你在回调函数中有必要调用 ``uv_freeaddrinfo``.

.. rubric:: dns/main.c
.. literalinclude:: ../code/dns/main.c
    :linenos:
    :lines: 41-59
    :emphasize-lines: 8,16

网络接口(Network interfaces)
----------------------------

系统网络接口信息可以通过调用 ``uv_interface_addresses`` 来获得,
下面的示例程序将打印出机器上所有网络接口的细节信息,
因此你可以获知网口的哪些域的信息是可以得到的, 这在你的程序启动时绑定 IP 很方便.

.. rubric:: interfaces/main.c
.. literalinclude:: ../code/interfaces/main.c
    :linenos:
    :emphasize-lines: 9,17

``is_internal`` 对于回环接口来说为 true. 请注意如果物理网口使用了多个
IPv4/IPv6 地址, 那么它的名称将会被多次报告, 因为每个地址都会报告一次.

.. _c-ares: http://c-ares.haxx.se
.. _getaddrinfo: http://www.kernel.org/doc/man-pages/online/pages/man3/getaddrinfo.3.html

.. _User Datagram Protocol: http://en.wikipedia.org/wiki/User_Datagram_Protocol
.. _DHCP: http://tools.ietf.org/html/rfc2131

----

.. [#] http://beej.us/guide/bgnet/output/html/multipage/advanced.html#broadcast
.. [#] 在 Windows 平台上仅支持 Windows Vista 以上的版本.
.. [#] http://www.tldp.org/HOWTO/Multicast-HOWTO-6.html#ss6.1
.. [#] libuv 使用线程池调用系统的 ``getaddrinfo`` 函数. libuv
    v0.8.0 及以前的版本引入 c-ares_ 作为异步 DNS 解析器, 但是在 v0.9.0中已经移除 c-ares_ 相关依赖了.
