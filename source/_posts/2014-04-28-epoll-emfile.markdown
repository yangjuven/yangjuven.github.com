---
layout: post
title: "epoll由EMFILE引发CPU飙升"
date: 2014-04-28 08:02
comments: true
categories: 
- epoll
- emfile
---

#### 问题表现

网易UU有一台加速节点，钊文发现突然 cpu 和 load average 飙升。最后曹局查到原因，是因为服务器系统升级重启后，单个进程打开的最大描述符数未设置好，被修改成1024了，钊文改回65536，重启服务，问题修复。

#### 追本溯源

这个问题很有意思，为何我们常用的 nginx 也遇到了 `Too many open files` 的问题，未见cpu飙升，为何加速节点上的服务会导致cpu飙升？咨询了下曹局，曹局解释说：

> 由于tcp监听队列满了，而异步io持续触发去读时又没有句柄能分配给这个网络请求，所以，导致队列一直是满的，但是因为没有fd而无法将请求从监听队列中移除去。

写个提供 echo 服务的 tcp server 验证下，很容易。

- [server](https://github.com/yangjuven/epoll-emfile/blob/test/server.py) 。使用 epoll 来进行多路IO复用。启动服务前，需要 `ulimit -n 20` 限制服务器能够打开的最大文件描述符数是 20，这样除了标准输入、标准输出、标准错误、监听socket对象、epoll对象占用了5个文件描述符，最多并发能够接受15个链接。
- [clinet](https://github.com/yangjuven/epoll-emfile/blob/test/client.py) 。模拟客户端发起16个并发链接。

大量的抛出 `Too many open files` 错误。

~~~ python
while True:
    events = epoll.poll()
    for fileno, event in events:
        if fileno == sock.fileno():
            try:
                connection, address = sock.accept()
                connection.setblocking(0)
                epoll.register(connection.fileno(), select.EPOLLIN | select.EPOLLOUT)
                connections[connection.fileno()] = connection
                packets[connection.fileno()] = b''
            except socket.error, ex:
                if ex.errno != errno.EMFILE:
                    raise
                logging.error(ex.strerror)
        elif ...
~~~


由于文件描述符的打开数量达到上限后， `accept` 的时候，抛出 `EMFILE` 错误，对于这个错误简单的处理方式是记下log，这样监听sock对象的 EPOLLIN 时间依然会被触发，陷入了死循环。因此cpu飙升。

#### 根本解决

该如何根本解决这个问题呢？我还曾经想过，能不能使用边缘触发呢？仅仅触发一次？

~~~ python
epoll.register(sock.fileno(), select.EPOLLIN | select.EPOLLET)
~~~
    
由于边缘触发，对于事件通知仅仅通知一次，为了防止丢事件，就必须一直重试。

>  In edge-triggered mode the program would need to accept() new socket connections until a socket.error exception occurs. 

给下代码或许清晰：
 
~~~ python
while True:
    events = epoll.poll()
    for fileno, event in events:
        if fileno == sock.fileno():
            while True:
                try:
                    connection, address = sock.accept()
                    connection.setblocking(0)
                    epoll.register(connection.fileno(), select.EPOLLIN | select.EPOLLOUT)
                    connections[connection.fileno()] = connection
                    packets[connection.fileno()] = b''
                except socket.error, ex:
                    if ex.errno not in (errno.EMFILE, errno.EAGAIN):
                        raise
                    logging.error(ex.strerror)
        elif ...
~~~

依然会陷入死循环。最好的解决方法是， **accept后立马关闭该连接** 。但是文件描述符都达到了上限，又accept不了。所以 **需要在服务器启动之初，就申请一个限制的文件描述符** ，当出现 `Too many open files` ，释放掉这个文件描述符，accept连接，接着立马关闭该连接。看代码

~~~ python
idle_fd = open('/dev/null')
...
while True:
    events = epoll.poll()
    for fileno, event in events:
        if fileno == sock.fileno():
            try:
                connection, address = sock.accept()
                connection.setblocking(0)
                epoll.register(connection.fileno(), select.EPOLLIN | select.EPOLLOUT)
                connections[connection.fileno()] = connection
                packets[connection.fileno()] = b''
            except socket.error, ex:
                if ex.errno != errno.EMFILE:
                    raise
                idle_fd.close()
                connection, address = sock.accept()
                connection.close()
                logging.error(ex.strerror)
                idle_fd = open('/dev/null')
~~~

问题得到完美解决。

#### References & Resoures

1. [How To Use Linux epoll with Python](http://scotdoyle.com/python-epoll-howto.html)
