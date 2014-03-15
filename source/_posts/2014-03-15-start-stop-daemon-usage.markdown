---
layout: post
title: "用start-stop-daemon将程序变为守护进程"
date: 2014-03-15 16:43
comments: true
categories: 
- daemon
---

在我们的工作中，接触了很多守护进程（daemon），比如 Web Server （Apache，Nginx），MySQL，Redis，Memcached 等等。除了这些开源程序，我们自己也会开发一些守护进程以满足业务的需要，比如UU加速节点上的 Echo Server 用来给玩家客户端提供测速服务。那么此时，我们需要特别关心的是：一个完整的守护进程需要满足哪些特性，以及如何实现这些特性。

首先谈谈我们对于守护进程的要求和期望的特性：

* **后台** 运行。
* 不会随着创建该守护进程的会话退出后，守护进程也跟着退出，要能 7x24 小时运行哇！
* 不能具有控制终端。杜绝从控制终端接收标准输入，还输出日志到控制终端。

在 《Advanced Programming in the UNIX Envrioment》一书中的 Chapter 13.Daemon Process ，就详细介绍 daemon 的编程规则和实现：

* 调用 `fork` 后，主进程退出，子进程忽略HUP信号。这样不仅能后台运行，还能忽略HUP信号，保证 7x24 小时运行。
* 调用 `setsid` 以创建一个新会话，使得调用进程：
  * 成为新会话的首进程
  * 成为新进程组的组长进程
  * 没有控制终端
  
这些操作可以使得一个程序满足了我们对于守护进程的期望，但是还远远不够，还需要：

* 调用 `umask` 设置权限掩码，保证守护进程创建新文件的权限。
* 调用 `chdir` 设置守护进程的工作目录。
* 调用 `setuid` 和 `setgid` 设置守护进程的用户。
* 关闭从父进程继承的不再需要的文件描述符。

如果在我们的代码中去实现一个守护进程，确实费心费力。所以大家都在寻找如何将我们的一个简单的程序变成守护进程？在UU加速节点中，启动 Echo Server 守护进程时，比较粗暴的通过以下命令：

~~~~ shell
nohup command 2>&1 >> log &
~~~~
    
这样的命令仅仅实现了后台运行和不随会话退出而提出，不仅很多细节没有实现，并且不够优雅。在 Debian 系统中， `start-stop-daemon` 就是为将一个普通程序变成守护进程而生。

* `-b, --background` 通过 fork 和 setsid 的形式将程序变为后台运行。
* `-d, --chdir` 更改进程的工作目录。
* `-u, --user` 设置进程的执行用户。
* `-k, --umask` 设置新建文件的权限掩码。

除此之外， start-stop-daemon 还可以

* `-S, --start` 启动程序
* `-K, --stop` 给程序发信号，终止程序或者判断程序的状态都可以

并且还通过 `-p, --pidfile` 和 `-m, --make-pidfile` 在启动程序时将守护进程启动后的 pid 写入指定文件，方便后续的终止程序或者判断程序的状态。

因为有了 `start-stop-daemon` 可以很容易写出系统启动脚本，网上的例子很多，比如这个 [模板](https://gist.github.com/alobato/1968852) ，其实：

* nginx /etc/init.d/nginx
* Redis /etc/init.d/redis-server

都是通过 start-stop-daemon 实现的，也是很好的模板。 start-stop-daemon 简单实用，赞一个！
