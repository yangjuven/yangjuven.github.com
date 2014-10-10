---
layout: post
title: "Web worker耗尽原因定位"
date: 2014-10-10 08:35
comments: true
categories: 
- linux
- nginx
- apache
- uwsgi
- tcp
- netstat
---

在我们的 Web 服务器中，当我们接收到服务器短信报警 LVS 监控 Real Server offline 的时候，你的第一反应会是什么？我一般都会从以下几个方面来诊断 offline 的真实原因：

1. 机器是否死机。
2. CPU 负载是否很高。
3. 内存是否不足。
4. 磁盘 IO 是否过高。
5. 网络是否有问题。
6. 操作系统资源限制，比如 `open file limit` 或者 `ip_conntrack: table full, dropping packet` 。
7. worker 资源被耗尽。

相信大家都遇到过类似的问题，并且对于前6种都可以轻松的诊断并识别。本文的重点主要是： **如何识别 worker 资源被耗尽，并且被耗尽的真实原因是什么？** 

#### worker定义

worker 是执行一个 HTTP 请求的最小执行单元，可以是 uWSGI 中的一个进程或者一个线程，也可以是 gevent 中的一个协程。worker 的数量一般都在配置文件中设置好，都是有限制的，一个 worker 在同一时间只能处理一个 HTTP 请求。一般情况下，worker 的数量表明同一时刻可以被处理的 HTTP 请求最高并发量。

nginx 中也有一个 worker 概念，这里的 worker 不同，是对应一个进程，由于采用多路 IO 复用，所以每个 worker 进程可以并发处理很多个请求，本文中 worker 定义更类似于 nginx 中的 [worker\_connections](http://nginx.org/en/docs/ngx_core_module.html#worker_connections) 。请注意这里是类似，还是有不同的：

> It should be kept in mind that this number includes all connections (e.g. connections with proxied servers, among others), not only connections with clients.

Apache 如果采用多进程模型，worker 的数量是等同于 `mpm_prefork_module` 中的 `MaxClients` 。如果采用的是多线程多进程模型，worker 的数量就等同于 `mpm_worker_module` 中的 `min(MaxClients, ThreadLimit * ThreadsPerChild)` 。

> prefork MPM
> 
> MaxClients: maximum number of server processes allowed to start.
> 
> worker MPM
> 
> MaxClients: maximum number of simultaneous client connections
> 
> ThreadLimit: ThreadsPerChild can be changed to this maximum value during a
>              graceful restart. ThreadLimit can only be changed by stopping
>              and starting Apache.
>              
> ThreadsPerChild: constant number of worker threads in each server process

uWSGI 如果是多进程多线程模型，worker 的数量等同于 [processes](http://uwsgi-docs.readthedocs.org/en/latest/Options.html#processes) * [threads](http://uwsgi-docs.readthedocs.org/en/latest/Options.html#threads) 。如果你的应用非线程安全，可以将 threads 设置为 1 。

mod\_wsgi 同理，也是 [processes * threads](https://code.google.com/p/modwsgi/wiki/QuickConfigurationGuide#Delegation_To_Daemon_Process) 。


#### worker被耗尽表现

在 worker 定义已经提到，worker 的数量都是有限制的，既然有限制，就会被耗尽。不论我们采用何种架构：

* Apache + mod\_wsgi
* Apache + fastcgi
* nginx + uWSGI (+ gevent)

由于木桶效应，不论是 Web Server 还是 App Server，有一个 worker 被耗尽，都会造成服务不可用。

如果 Apache 的 worker 资源好耗尽，造成新来的 HTTP 链接无法及时被 accept ，一直到塞满 backlog 。在 《Unix Network Programming》 Section 4.5 'listen' Function 提到：

> int listen (int sockfd, int backlog);
>
> To understand the backlog argument, we must realize that for a given listening socket, the kernel maintains two queues:
>
> 1. An incomplete connection queue, which contains an entry for each SYN that has arrived from a client for which the server is awaiting completion of the TCP three-way handshake. These sockets are in the SYN\_RCVD state (Figure 2.4).
>
> 2. A completed connection queue, which contains an entry for each client with whom the TCP three-way handshake has completed. These sockets are in the ESTABLISHED state (Figure 2.4).

当这两个队列的总量超过 backlog 后，新来的 HTTP 连接三次握手的 SYN 包，服务器就不会理会了，直接丢弃。所以在客户端或者浏览器中的表现就是，一直等待。

如果 nginx 的 worker\_connections 被耗尽，nginx error log 会报错 `worker_connections is not enough`，至于被耗尽时 nginx 如何处理的，还是由顾爷来分析吧。

而对于后端的 App Server 如果 worker 会耗尽，Web Server 接受到上游 App Server 的返回错误，比如 `Resource temporarily unavailable` ，一般都会返回给客户端一个 `502 Bad Gateway` 。

#### 定位真实原因

如果你发现 worker 被耗尽，觉得是 worker 数量太少导致的，所以立马修改配置提升 worker 数量？这样就大错特错了，在没有搞清楚原因前，盲目提高 worker 数量，在新来的连接面前杯水车薪，不堪一击。对于我们的 Web 应用，在流量没有出现井喷的情况下，没有被 DDoS ，worker 被耗尽，一般都是 worker 处理一个 HTTP 请求的响应时间增多。而我们的应用大部分应用都是 IO bound ，并且是网络 IO 密集型，处理 HTTP 请求的响应时间时间增多，一般都是网络请求处理时间长了。定位到处理时间变长的网络请求，一般就可以定位到真实原因了。在 App Server 中的网络请求大致分为两种：

- 长连接请求。比如连接复用 MySQL 连接。如果核实这类网络请求的耗时是否增加，可以登录到 MySQL 服务器，依然先从内存、CPU、IO 判别，再接着通过 `show processlist;` 来看当前有哪些阻塞的请求，一目了然。
- 短连接请求。比如 HTTP 接口调用，未复用连接的 MySQL、Redis 连接。

对于短连接请求，TCP 三次握手连接建立后，立马请求所需，完成后立马关闭。如果这类请求的耗时增加，完全可以通过 netstat 命令捕获到。我一般都是通过以下这种命令来定位的：

~~~ shell
netstat -ant | awk '{if($4 !~ /:80$/ && $6 != "LISTEN" && $6 != "TIME\_WAIT") print $5;}' | \
    sort | uniq -c | awk 'BEGIN {OFS="\t"} {print $1,$2;}' | sort -nr -k 1,1 | \
    awk 'BEGIN {OFS="\t"} {if($1 != 1) print $2,$1;}'
~~~
        
给大家解释下这个 shell 命令：

1. 先获取当前系统的 TCP 连接，从中过滤掉这些连接：
   - 客户端发过来的 HTTP 请求。
   - 处于 LISTEN 状态的连接。
   - 处于 TIME\_WAIT 状态的连接。为何过滤掉 TIME\_WAIT ？请看我以前的这篇文章 [深入理解TCP的TIME-WAIT](/blog/2014/06/11/tcp-time-wait/) 。
2. 根据 TCP 连接的目的地址目的端口进行汇总，并且根据汇总量进行排序，排除总量是1的。

通过这个命令看出当前占用比较多的连接数，排除那些连接复用的，剩下短连接，如果短连接数量比较多，接近 worker 数量，就得注意了。当你发现可疑的目的地址端口，可以 `nslookup <ip>` 这样获取域名，你一般就是是哪个接口出问题啦。
