---
layout: post
title: Tornado 源码分析 - 基础篇
meta: tornado source code analysis - basic
draft: false
category: [tornado, reactor]
---

源码分析基于 [Tornado 2.0](https://pypi.python.org/pypi/tornado/2.0) 版本，2.0 版本时候 Tornado 的介绍还是：

> Tornado is an open source version of the scalable, non-blocking web server and and tools that power FriendFeed.

这个时候介绍中还未加入 `asynchronous networking library`，关于 Tornado 在以后版本中对 asynchronous 的支持，在下篇文章会进行介绍。
这篇文章主要介绍 Tornado 基本执行流程，对 HTTP Request 的解析和处理，和 `No-blocking I/O` 在 Tornado 的具体应用。

## 执行流程

我们用一个简单的 Hello World 示例分析其基本的执行流程。

    #!/usr/bin/env python

    import tornado.httpserver
    import tornado.ioloop
    import tornado.options
    import tornado.web

    from tornado.options import define, options

    define("port", default=8888, help="run on the given port", type=int)


    class MainHandler(tornado.web.RequestHandler):
        def get(self):
            self.write("Hello, world")


    def main():
        tornado.options.parse_command_line()
        application = tornado.web.Application([
            (r"/", MainHandler),
        ])
        http_server = tornado.httpserver.HTTPServer(application)
        http_server.listen(options.port)
        tornado.ioloop.IOLoop.instance().start()


    if __name__ == "__main__":
        main()

代码测试无误后，添加 `pdb.set_trace()` 开始获取 "调用堆栈"。

    class TestHandler(RequestHandler):
        def get(self):
            import pdb
            pdb.set_trace()
            self.write("Hello, World!\n")

使用 `curl` 请求 `curl http://localhost:8888` 触发断点。
在 `pdb` 中输入 `w` 获取详细堆栈信息，输入 `u` `d` 可在上下堆栈之间切换，查看函数的调用情况。
以上测试方法来自于 [参考资料 2](http://blog.csdn.net/zhaoxia_guo/article/details/6921572)。

### IOLoop

先来看一下 IOLoop 的基本功能：
![IOLoop 的基本功能](/assets/images/tornado/ioloop.png)
可以看到 IOLoop 是典型的 [Reactor 模型](https://en.wikipedia.org/wiki/Reactor_pattern#Structure) 的实现。

再来看一下 Hello World 那个代码的详细执行流程，以及 IOLoop 是如何发挥总调度作用的：
![IOLoop 调度流程](/assets/images/tornado/ioloop-scheduler.png)
比较重要的是绿色部分把 listening socket fd 和 connecting socket fd 添加到 ioloop 中的过程。然后 ioloop 在监听到这些 fd 可用之后，开始调用对应的处理函数开始处理。

### HTTP Request 的解析和处理

![HTTP Request 的解析和处理](/assets/images/tornado/http-request-parser.png)

数据的读写都是通过 IOStream 实现，绿色部分表示对 HTTP Request Header 和 HTTP Request Body 的处理。

为加深对 IOLoop 的理解，我们来实现一个简单的 EchoServer。

    def handle_accept(fd, events):
        connection, address = fd_sockets[fd].accept()
        fd_sockets[connection.fileno()] = connection

        io_loop = tornado.ioloop.IOLoop.instance()
        io_loop.add_handler(connection.fileno(), handle_read, io_loop.READ)


    def handle_read(fd, events):
        data = fd_sockets[fd].recv(1024)
        if data:
            fd_sockets[fd].sendall(data)
        else:
            io_loop = tornado.ioloop.IOLoop.instance()
            io_loop.remove_handler(fd)
            fd_sockets[fd].close()


    def main():
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.bind((HOST, PORT))
        s.listen(5)
        fd_sockets[s.fileno()] = s

        io_loop = tornado.ioloop.IOLoop.instance()
        io_loop.add_handler(s.fileno(), handle_accept, io_loop.READ)
        io_loop.start()

### 各 Class 的主要作用

Tornado 各个类的分工还是很清晰的，实现也比较容易看懂。

- HTTPServer

    处理套接字的 listen/bind/accept。

- IOStream

    处理套接字的 read/write。

- HTTPConnection

    处理与 HTTP client 建立的连接，解析 HTTP Request 的 header 和 body。

- IOLoop

    I/O loop，循环取出可用的 fd，并调用对应的事件处理函数。

- RequestHandler

    处理请求，支持 GET/POST 等操作。

## No-bloking I/O

先看两个事件处理函数：

    def handle_read():
        socket.setblocking(False)
        while True:
            try:
                data = socket.recv(1024)
            except socket.error, e:
                if e.args[0] in (errno.EWOULDBLOCK, errno.EAGAIN):
                    return
                raise


    def handle_accept():
        socket.setblocking(False)
        while True:
            try:
                connection, address = sockets.accept()
            except socket.error, e:
                if e.args[0] in (errno.EWOULDBLOCK, errno.EAGAIN):
                    return
                raise

考虑一下这里为什么要采用非阻塞方式来 read/accept 数据？
我们发现 IOLoop 已经通过多路复用帮我们监听了所有 fds，等 fd 可用之后才开始调用对应的事件处理函数。换句话说在调用 handle_read/handle_accept 我们已经知道 fd 可用了，那为什么不采用阻塞 I/O 读取，而要采用非阻塞 I/O 的方式呢？
问题其实可以抽象成「**多路复用为什么要搭配非阻塞 I/O**」，具体讨论可以参考 [这里](https://www.zhihu.com/question/37271342/answer/81607536)。

## 参考资料

* [Tornado 2.0](https://pypi.python.org/pypi/tornado/2.0)
* [有没有什么很好的 Tornado 的教材或者开源项目可以做参考的？](https://www.zhihu.com/question/19707966)
* [Tornado: 1. 流程分析](http://blog.csdn.net/zhaoxia_guo/article/details/6921572)
* [Tornado: 2. 源码分析](http://blog.csdn.net/zhaoxia_guo/article/details/6925811)
* [高性能网络编程（一）- accept 建立连接](http://blog.csdn.net/russell_tao/article/details/9111769)
* [为什么 IO 多路复用要搭配非阻塞 IO?](https://www.zhihu.com/question/37271342)