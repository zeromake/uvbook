序言
====

本书由一系列 libuv_ 教程组成， libuv_
是一个高性能事件驱动的程序库，封装了 Windows 和 Unix 平台一些底层特性，为开发者提供了统一的 API.

本书旨在涵盖 libuv 的主要特性, 并不是一份完整介绍 libuv 内部每个 API 和数据结构的指南,
官方文档 `official libuv documentation`_ 可以直接在 libuv 源码提供的头文件中找到.

.. _official libuv documentation: https://github.com/joyent/libuv/blob/master/include/uv.h

本书还没有完成，某些章节可能不完整，但我希望在我不断完善本书同时，你也能够从中获益 :-)

本书为谁而写?
-------------

如果你正在阅读本书，你或许是:

1) 系统开发人员, 编写一些诸如守护进程(daemons), 网络服务程序或者客户端等底层应用,
   你发现 libuv 的事件循环方式适合你的应用场景, 因此你决定使用 libuv.

2) Node.js 某一模块的作者, 决定使用 C/C++ 封装系统平台某些同步或者异步 API,
   并将其暴露给 Javasript, 你可以在 node.js 上下文中只使用 libuv,
   但你也需要参考其他资源, 因为本书并没有包括 v8/node.js 相关的内容.

本书假设你对 C 语言有了一定的了解。

背景
----

node.js_ 最初发起于 2009 年, 是一个可以让 Javasript 代码脱离浏览器的执行环境,
libuv 使用了 Google 的  V8_ 执行引擎 和 Marc Lehmann 的 libev_. Node.js
将事件驱动的 I/O 模型与适合该模型的编程语言(Javasript)融合在了一起,
随着 node.js 的日益流行, node.js 的开发者们也意识到应该让 node.js
在 Windows 平台下也能工作, 但是 libev 只能在 Unix 环境下运行.
Windows 平台上与 kqueue(FreeBSD) 或者 (e)poll(Linux) 等内核事件通知相应的机制
是 IOCP, libuv 依据不同平台的特性(Unix 平台为 libev, Windows 平台为 IOCP)
给上层应用提供了统一基于 libev API 的抽象,
不过 node-v0.9.0 版本的 libuv 中 libev 的依赖已被移除, 参见: `libev has been removed`_
libuv 直接与 Unix 平台交互.

本书代码
--------

本书所有代码均可以在 Github 上获取, `Clone`_/`Download`_ 本书源码，然后进入到
``code/`` 目录执行 ``make`` 编译本书的例子. 书中的代码基于 `node-v0.9.8`_ 版本的 libuv,
为了方便读者学习，本书的源码中也附带了相应版本的 libuv，你可以在 ``libuv/``
目录中找到源码，libuv 会在你编译书中的例子时被自动编译。

.. _Clone: https://github.com/forhappy/uvbook
.. _Download: https://github.com/forhappy/uvbook/downloads
.. _node-v0.9.8: https://github.com/joyent/libuv/tags
.. _V8: http://code.google.com/p/v8/
.. _libev: http://software.schmorp.de/pkg/libev.html
.. _libuv: https://github.com/joyent/libuv
.. _node.js: http://www.nodejs.org
.. _libev has been removed: https://github.com/joyent/libuv/issues/485
