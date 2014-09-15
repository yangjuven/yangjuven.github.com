---
layout: post
title: "Linux中SO_REUSEADDR和SO_REUSEPORT区别"
date: 2014-09-14 18:45
comments: true
categories: 
- Linux
- socket
- tcp
---

最近在看技术文档中，经常出现 `SO_REUSEADDR` 和 `SO_REUSEPORT` 两个 socket option ，这两者的区别仅从字面意义无法区别，比较含糊、模糊不清。今天我们就详述下 **Linux** 中两者的区别。（请注意：是 Linux 系统，其他系统就不比较了。如果你想全面了解各个系统中的差异，可以看 [这里](http://stackoverflow.com/questions/14388706/socket-options-so-reuseaddr-and-so-reuseport-how-do-they-differ-do-they-mean-t/14388707#14388707) ）

#### 基础知识

我们都知道，通过五元组（协议、源地址、源端口、目的地址、目的端口）可以 **唯一定位** 一个连接。通过 socket、bind、connect 三个系统调用可以为一个 socket 分配五元组。

* socket 中通过 SOCK\_STREAM(TCP)、SOCK\_DGRAM(UDP) 指定协议。
* bind 来绑定源地址、源端口。bind 这步也可以省略，如果省略，内核会为该 socket 分配一个源地址、源端口。另外，如果 bind 的源地址为 0.0.0.0，内核也会从本地接口中分配一个作为源地址。如果 bind 的源端口是 0，内核也会自动分配源端口。
* connect 来指定目的地址、目的端口。有同学会问，UDP 是无连接状态的，也能 connect 吗？当然可以的，详情请 `man connect` ，无非 UDP 在 connect 的时候没有三次握手嘛。如果 UDP 不通过 connect 来指定目的地址、目的端口，那发送数据包时，就必须使用 `sendto` 而不是 `send` ，在 sendto 的时候指定目的地址、目的端口。

为了能够通过五元组唯一定位一个连接，内核在给连接分配源地址、源端口的时候，必须使不同的连接具有不同的五元组。因此，在默认情况下，两个不同的 socket 不能 bind 到相同的源地址和源端口。


#### SO\_REUSEADDR

在有些情况下，这个默认情况下的限制: **两个不同的 socket 不能 bind 到相同的源地址和源端口** ，带来很大的困扰。

* [TCP 的 TIME\_WAIT 状态](http://blog.qiusuo.im/blog/2014/06/11/tcp-time-wait/) 时间过长，造成新 socket 无法复用这个端口，即使可以确定这个连接可以销毁。完全是 **拉完屎还占着茅坑** 。这个问题在重启 TCP Server 时更为严重。
* `0.0.0.0` 和本地接口地址，也会被认为是相同的源地址，从而 bind 失败。

在 unix 系统中，`SO_REUSEADDR` 就是为了解决这些困扰而生。 假设当前系统有两个本地接口，一个是 `192.168.0.1`，一个是 `10.0.0.1`，分别对 socketA 和 socketB 进行以下各种情况的绑定以及出现的结果：

<table>
    <thead>
        <tr>
            <th>SO_REUSEADDR</th>
            <th>socketA</th>
            <th>socketB</th>
            <th>Result</th>
        </tr>
    </thead>
    <tbody>
         <tr>
            <td>ON/OFF</td>
            <td>192.168.0.1:21</td>
            <td>192.168.0.1:21</td>
            <td>Error (EADDRINUSE)</td>
        </tr>
        <tr>
            <td>ON/OFF</td>
            <td>192.168.0.1:21</td>
            <td>10.0.0.1:21</td>
            <td>OK</td>
        </tr>
        <tr>
            <td>ON/OFF</td>
            <td>10.0.0.1:21</td>
            <td>192.168.0.1:21</td>
            <td>OK</td>
        </tr>
        <tr>
            <td>OFF</td>
            <td>0.0.0.0:21</td>
            <td>192.168.1.0:21</td>
            <td>Error (EADDRINUSE)</td>
        </tr>
        <tr>
            <td>OFF</td>
            <td>192.168.1.0:21</td>
            <td>0.0.0.0:21</td>
            <td>Error (EADDRINUSE)</td>
        </tr>
        <tr>
            <td>ON</td>
            <td>0.0.0.0:21</td>
            <td>192.168.1.0:21</td>
            <td>OK</td>
        </tr>
        <tr>
            <td>ON</td>
            <td>192.168.1.0:21</td>
            <td>0.0.0.0:21</td>
            <td>OK</td>
        </tr>
        <tr>
            <td>ON/OFF</td>
            <td>0.0.0.0:21</td>
            <td>0.0.0.0:21</td>
            <td>Error (EADDRINUSE)</td>
        </tr>
    </tbody>
</table>

但是在 Linux 中，在 `man socket` 中可以看到对 `SO_REUSEADDR` 的解释。

> Indicates that the rules used in validating addresses supplied in a bind(2) call should allow reuse of local addresses. For AF\_INET sockets this means that a socket may bind, except when there is an active listening socket bound to the address. When the listening socket is bound to INADDR\_ANY with a specific port then it is not possible to bind to this port for any local address. Argument is an integer boolean flag.

就是说，跟 unix 对比起来，有一个例外，如果对于监听 socket 来，如果已经 bind 到 0.0.0.0 ，其他监听 socket 就不能 bind 到任何一个本地接口了。

#### SO\_REUSEPORT

Linux 在内核 3.9 中添加了新的 socket option `SO_REUSEPORT` 。

> One of the features merged in the 3.9 development cycle was TCP and UDP support for the SO\_REUSEPORT socket option; that support was implemented in a series of patches by Tom Herbert. The new socket option allows multiple sockets on the same host to bind to the same port, and is intended to improve the performance of multithreaded network server applications running on top of multicore systems.

如果在 bind 系统调用前，指定了 SO\_REUSEPORT ，多个 socket 便可以 bind 到相同的源地址、源端口，比起 SO\_REUSEADDR 更强大、更劲爆，有木有。不仅如此，还添加了权限保护，为了防止 **端口劫持** ，在第一个 socket bind 成功后，后续的 socket bind 的用户必须或者是 root，或者跟第一个 socket 用户一致。

#### 使用 SO\_REUSEPORT 杜绝 accept 惊群

这里是本文的重点。以前的 TCP Server 开发中，为了充分利用多核的性能，所以在多进程中监听同一个端口。在没有 SO\_REUSEPORT 的时代，可以通过 fork 来实现。父进程绑定一个端口监听 socket ，然后 fork 出多个子进程，子进程们开始循环 accept 这个 socket 。但是会带来一个问题：如果有新连接建立，哪个进程会被唤醒且能够成功 accept 呢？在 Linux 内核版本 2.6.18 以前，所有监听进程都会被唤醒，但是只有一个进程 accept 成功，其余失败。这种现象就是所谓的 **惊群效应** 。其实在 2.6.18 以后，这个问题得到修复，仅有一个进程被唤醒并 accept 成功。 

但是，现在的 TCP Server，一般都是 `多进程+多路IO复用(epoll)` 的并发模型，比如我们常用的 nginx 。如果使用 epoll 去监听 accept socket fd 的读事件，当有新连接建立时，所有进程都会被触发。因为由于 fork 文件描述符继承的缘故，所有进程中的 accept socket fd 是相同的。惊群效应依然存在。nginx 也必然存在这个问题，nginx 为了解决问题，并且保证各个 worker 之前 accept 连接数的均衡，费了很大的力气。

有了 SO\_REUSEPORT ，解决 多进程+多路IO复用(epoll) 并发模型 accept 惊群问题，就简单、高效很多。我们不需要通过 fork 的形式，让多进程监听同一个端口。只需要在各个进程中， **独自的** 监听指定的端口，当然在监听前，我们需要为监听 socket 指定 SO\_REUSEPORT ，否则会报错啦。由于没有采用 fork 的形式，各个进程中的 accept socket fd 不一样，加之有新连接建立时，内核只会唤醒一个进程来 accept，并且保证唤醒的 **均衡性**，因此使用 epoll 监听读事件，就不会触发所有啦。也有牛人为 nginx 提了 [patch](http://lwn.net/Articles/542629/) ，使用 SO\_REUSEPORT 来杜绝 accept 惊群，并且还能够保证 worker 之间的均衡性哦。

因此使用 SO\_REUSEPORT ，可以在多进程网络并发服务器中，可以充分利用多核的优势。

#### References && Resources

* [Difference between SO\_REUSEADDR and SO\_REUSEPORT](http://stackoverflow.com/questions/14388706/socket-options-so-reuseaddr-and-so-reuseport-how-do-they-differ-do-they-mean-t/14388707#14388707)
* [The SO\_REUSEPORT socket option](http://lwn.net/Articles/542629/)
* [[PATCH] SO\_REUSEPORT support for listen sockets](http://forum.nginx.org/read.php?29,241283,241283)
