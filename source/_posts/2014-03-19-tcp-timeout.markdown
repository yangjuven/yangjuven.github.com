---
layout: post
title: "TCP协议的那些超时"
date: 2014-03-19 08:28
comments: true
categories: 
- tcp
---

众所周知，TCP协议是一个 **可靠的** 的协议。TCP的可靠性依赖于大量的 **Timer** 和 **Retransmission** 。现在咱们就来细说一下TCP协议的那些 **Timer** 。

![TCP State transition diagram](/images/tcp-state-transition.png)

#### 1. Connection\-Establishment Timer

在TCP三次握手创建一个连接时，以下两种情况会发生超时：

1. client发送SYN后，进入SYN\_SENT状态，等待server的SYN+ACK。
2. server收到连接创建的SYN，回应SYN+ACK后，进入SYN\_RECD状态，等待client的ACK。

当超时发生时，就会重传，一直到75s还没有收到任何回应，便会放弃，终止连接的创建。但是在Linux实现中，并不是依靠超时总时间来判断是否终止连接。而是依赖重传次数：

> * tcp\_syn\_retries (integer; default: 5; since Linux 2.2)
> 
>   The maximum number of times initial SYNs for an active TCP connection attempt will be retransmitted. This value should not be higher than 255. The default value is 5, which corresponds to approximately 180 seconds.
>   
> * tcp\_synack\_retries (integer; default: 5; since Linux 2.2)
> 
>   The maximum number of times a SYN/ACK segment for a passive TCP connection will be retransmitted. This number should not be higher than 255.
    


#### 2. Retransmission Timer

当三次握手成功，连接建立，发送TCP segment，等待ACK确认。如果在指定时间内，没有得到ACK，就会重传，一直重传到放弃为止。Linux中也有相关变量来设置这里的重传次数的：

> * tcp\_retries1 (integer; default: 3; since Linux 2.2)
> 
>   The number of times TCP will attempt to retransmit a packet on an established connection normally, without the extra effort of getting the  network  layers  involved. Once  we exceed this number of retransmits, we first have the network layer update the route if possible before each new retransmit.  The default is the RFC specified minimum of 3.
> 
> * tcp\_retries2 (integer; default: 15; since Linux 2.2)
> 
>   The maximum number of times a TCP packet is retransmitted in established state before giving up. The default value is 15, which corresponds to a duration of approxi‐mately between 13 to 30 minutes, depending on the retransmission timeout.  The RFC 1122 specified minimum limit of 100 seconds is typically deemed too short.


#### 3. Delayed ACK Timer

当一方接受到TCP segment，需要回应ACK。但是不需要 **立即** 发送，而是等上一段时间，看看是否有其他数据可以 **捎带** 一起发送。这段时间便是 **Delayed ACK Timer** ，一般为200ms。

#### 4. Persist Timer

如果某一时刻，一方发现自己的 socket read buffer 满了，无法接受更多的TCP data，此时就是在接下来的发送包中指定通告窗口的大小为0，这样对方就不能接着发送TCP data了。如果socket read buffer有了空间，可以重设通告窗口的大小在接下来的 TCP segment 中告知对方。可是万一这个 TCP segment 不附带任何data，所以即使这个segment丢失也不会知晓（ACKs are not acknowledged, only data is acknowledged）。对方没有接受到，便不知通告窗口的大小发生了变化，也不会发送TCP data。这样双方便会一直僵持下去。

TCP协议采用这个机制避免这种问题：对方即使知道当前不能发送TCP data，当有data发送时，过一段时间后，也应该尝试发送一个字节。这段时间便是 Persist Timer 。

#### 5. Keepalive Timer

TCP socket 的 SO\_KEEPALIVE option，主要适用于这种场景：连接的双方一般情况下没有数据要发送，仅仅就想尝试确认对方是否依然在线。目前vipbar网吧，判断当前客户端是否依然在线，就用的是这个option。

具体实现方法：TCP每隔一段时间（tcp\_keepalive\_intvl）会发送一个特殊的 Probe Segment，强制对方回应，如果没有在指定的时间内回应，便会重传，一直到重传次数达到 tcp\_keepalive\_probes 便认为对方已经crash了。

> * tcp\_keepalive\_intvl (integer; default: 75; since Linux 2.4)
> 
>   The number of seconds between TCP keep-alive probes.
> 
> * tcp\_keepalive\_probes (integer; default: 9; since Linux 2.2)
> 
>   The maximum number of TCP keep-alive probes to send before giving up and killing the connection if no response is obtained from the other end.
> 
> * tcp\_keepalive\_time (integer; default: 7200; since Linux 2.2)
> 
>   The number of seconds a connection needs to be idle before TCP begins sending out keep-alive probes.  Keep-alives are only sent when the SO\_KEEPALIVE socket option is enabled.  The default value is 7200 seconds (2 hours).  An idle connection is terminated after approximately an additional 11 minutes (9 probes an interval of 75 sec‐onds apart) when keep-alive is enabled.
> 
> Note that underlying connection tracking mechanisms and application timeouts may be much shorter.


#### 6. FIN\_WAIT\_2 Timer

当主动关闭方想关闭TCP connection，发送FIN并且得到相应ACK，从FIN\_WAIT\_1状态进入FIN\_WAIT\_2状态，此时不能发送任何data了，只等待对方发送FIN。可以万一对方一直不发送FIN呢？这样连接就一直处于FIN\_WAIT\_2状态，也是很经典的一个DoS。因此需要一个Timer，超过这个时间，就放弃这个TCP connection了。

> * tcp_fin_timeout (integer; default: 60; since Linux 2.2)
> 
>   This specifies how many seconds to wait for a final FIN packet before the socket is forcibly closed.  This is strictly a  violation  of  the  TCP  specification,  but required to prevent denial-of-service attacks.  In Linux 2.2, the default value was 180.


#### 7. TIME\_WAIT Timer

TIME\_WAIT Timer存在的原因和必要性，主要是两个方面：

1. 主动关闭方发送了一个ACK给对方，假如这个ACK发送失败，并导致对方重发FIN信息，那么这时候就需要TIME\_WAIT状态来维护这次连接，因为假如没有TIME\_WAIT，当重传的FIN到达时，TCP连接的信息已经不存在，所以就会重新启动消息应答，会导致对方进入错误的状态而不是正常的终止状态。假如主动关闭方这时候处于TIME\_WAIT，那么仍有记录这次连接的信息，就可以正确响应对方重发的FIN了。
2. 一个数据报在发送途中或者响应过程中有可能成为残余的数据报，因此必须等待足够长的时间避免新的连接会收到先前连接的残余数据报，而造成状态错误。

但是我至今疑惑的是：为什么这个超时时间的值为2MSL？如果为了保证双方向的TCP包要么全部响应完毕，要么全部丢弃不对新连接造成干扰，这个时间应该是：

> 被动关闭方LAST\_ACK的超时时间 + 1MSL

因为被动关闭方进入LAST\_ACK状态后，假设一直没有收到最后一个ACK，会一直重传FIN，一直重传次数到达TCP\_RETRIES放弃，将这个时间定义为「被动关闭方LAST\_ACK的超时时间」，接着必须等待最后一个重传的FIN失效，需要一个MSL的时间。这样才能保证所有重传的FIN包失效，不干扰新连接吧。


References & Resources

1. TCP/IP Illustrated
