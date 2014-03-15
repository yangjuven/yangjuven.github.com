---
layout: post
title: "UDP server绑定IP到INADDR_ANY？"
date: 2014-03-15 16:47
comments: true
categories: 
- udp
- tcp
- ip
- java
- echo
---

#### 背景介绍

玩家在使用UU加速器，智能选择最佳加速节点时，是需要进行测速，一般都选择ping，用RTT（往返时延）来衡量网络环境的优差。但是有些玩家的网络环境封锁了icmp协议，此时就需要通过加速节点上的 Echo 服务进行测试了。UU在每个加速节点都部署有 Echo 服务，就是客户端发个ping包，服务端回个pong包，主要也是用来测试往返时延。目前的 Echo 服务是启了一个 UDP Server ，收包回包。

#### 问题

我刚接触这个问题，是SA同学提出的。钊文在部署新加速节点，如果该节点是双线（一电信IP，一联通IP）的话，需要启动两个 Echo 服务实例，一个实例绑定一个IP。带来了两点麻烦：

* 部署新节点比较麻烦，需要手动修改启动命令
* 一个实例可以搞定的事儿，非得启动两个，对于内存消耗也不少，目前线上服务器每个 Echo 实例消耗的内存在 200-300 M

因此，这次任务的目标是： **在服务器上启动一个 Echo 实例** 。

#### 深入

当时我很纳闷，在代码中，UDP server 启动时， socket `bind` 到 `0.0.0.0` 即可吧，所有 interface 都可以接包响应服务。在 [ip - Linux IPv4 protocol implementation](http://man7.org/linux/man-pages/man7/ip.7.html) 阐述的很清楚

> When a process wants to receive new incoming packets or connections,
> it should bind a socket to a local interface address using bind(2).
> In this case, only one IP socket may be bound to any given local
> (address, port) pair.  When INADDR\_ANY is specified in the bind call,
> the socket will be bound to all local interfaces.

我读了 Echo Server 的代码，发现代码中确实可以以 `bind` 到 `0.0.0.0` 的形式启动。因此在我本地 mac 上，启动这个 Echo 服务，并且通过自写的客户端通过以下三个ip发起echo测速。

1. lo 127.0.0.1
2. en0 192.168.224.28 有线连接
3. en1 10.255.201.235 wifi连接

都能正常接收到pong包。但是当我在备用加速节点 xa1\_tel 服务器上进行测试时，该服务器有两个外网IP：

1. eth0 117.xx.xx.140
2. eth1 123.xxx.xx.73

在服务器上启动 Echo 服务，在我本地发起 Echo 测速，电信IP是可以进行正常 echo 的，但是和联通IP收不到服务器返回的pong包。通过在服务器上 `sudo tcpdump -i any port 9999` 抓包发现：

~~~~ shell
18:01:42.794450 IP 218.xxx.xx.253.58971 > 123.xxx.xx.73.9999: UDP, length 2
18:01:40.211172 IP 117.xx.xx.140.9999 > 218.xxx.xx.253.58971: UDP, length 2
~~~~

也就是说，发给联通IP的ping包，返回的pong包通过电信IP发出了。由于IP的改动，五元组（协议，源IP，源端口，目的IP，目的端口）都变化了，造成我本地的客户端在应用层接受不到数据了。

#### 原因

和曹局咨询了原因，以及曹局推荐我看了这篇文章 [Linux路由应用-使用策略路由实现访问控制](http://www.oschina.net/question/234345_47473) ，得知，UDP 和 TCP 在 bind 有很大不同：

1. TCP 是面向连接，可靠的，Linux内核维持TCP连接时，必然保存了五元组。即使 bind 到 0.0.0.0 ，其ip层的源地址，是由tcp层来确定。
2. UDP是不可靠，无连接，对于源ip和目的ip的管理很松散，很飘。如果 bind 到 0.0.0.0 ，在服务器回送pong包时，其源地址便于路由来决定了。因为，选择源地址原则是：优先选择和下一跳IP地址为同一网段的 interface ip ，而下一跳地址是由路由决定的。

为什么测试本地的 Echo 服务正常，而测试 xa1\_tel 就不行呢？有了上面的第2条原则，解释这个就不难。在本地测试 Echo 服务时，不论UDP包目的IP是哪个IP，下一跳IP地址同一个网段的 interface IP 必然是其自身。但是在 xa1\_tel 测试时，猜测有这样的路由

~~~~ shell
218.xxx.xx.253 gw 117.xx.xx.191
~~~~

因此当 xa1\_tel 发回pong包时，并且还是 bind 到 0.0.0.0 ，只要ping包的来源IP是 218.xxx.xx.253 ，此时选择的路由便是通过电信网关发送，因此源IP也被设置成了电信IP。

#### 解决

知道原因，解决问题就很简单了。获取服务器所有 interface IP （由于UU要求，还需要排除 lo 和 虚拟网卡IP），遍历 bind 一次即可。但是事情进展没有那么顺利，还有一些小波澜。Echo Server 是用 java 写的。我通过这个语句获取所有 interface ：

~~~~ java
Enumeration<NetworkInterface> interfaces = NetworkInterface.getNetworkInterfaces();
~~~~
    
在有些测试服务器上运行OK，在有些服务器上抛出错误：

~~~~ shell
*** glibc detected *** /usr/bin/java: malloc(): memory corruption: 0x00007f153009fb30 ***
~~~~
    
我当时吓尿了，第一次写出了 memory corruption 的代码。研究半天，觉得是java的一个 Bug ：[JDK-7078386 : NetworkInterface.getNetworkInterfaces() may return corrupted results on linux](http://bugs.java.com/bugdatabase/view_bug.do?bug_id=7078386) 

> A DESCRIPTION OF THE PROBLEM :
> calling NetworkInterface.getNetworkInterfaces() on linux returns corrupted results if some interface's index is over 255 (which is sometimes the case for virtual interfaces).

UU的加速服务，虚拟网卡确实比较多， index 确实有超过 255 ，而这个 Bug 是在 jdk8 中才修复，我只有写个 python 脚本解析 `ip addr` 获取所有 interface IP，传递给 Echo Server 。
