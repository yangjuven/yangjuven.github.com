---
layout: post
title: "删除守护进程的日志"
date: 2014-08-18 16:55
comments: true
categories: 
- Linux
- fs
---

跟我一起面试过的同学，或者被我面试过的同学，应该都知道，我在面试中为了更加深入的了解候选同学对于 Linux 文件系统掌握情况，其中经常问的一道题目就是： **删除一个守护进程的日志文件，会发生什么？** 最近，在工作中也遇到类似问题，我就发篇 blog 总结下哈。

#### 问题表现

我所在项目组的测试服务器连续发生两次硬盘空间报警，都是因为 /tmp/mysql_query.log 过大。
不过这不是我今天的主题。有意思的地方在于，出现这种情况后，为了及时清理出硬盘空间，如果简单执行 `rm /tmp/mysql_query.log` ，发现报警不会解决，通过 df 命令发现硬盘空间占用依然超90%，只有重启 mysql 才得以清理出硬盘空间，报警得以解决。

#### 分析原因

顾爷也在群里贴出了原因：

> The df command reports the number of disk blocks used while du goes through the
> file structure and and reports the number of blocks used by each directory.  As
> far as du is concerned, the file used by the process does not exist, so it does
> not report blocks used by this phantom file.  But df keeps track of disk blocks
> used, and it reports the blocks used by this phantom file.

恩，出现了 `phantom file` ，至于为何出现 phantom file。需要从文件系统的本质说起。

![Cylinder groups's i-nodes and data blocks in more detail](/images/inode.gif)

此图摘自 《Advanced Programming in the UNIX Environment》4.14 File Systems 。细节我就不解释啦。

每个 `inode` 结点存储了关于文件的元信息，比如指向 data block 的指针。在本文中，息息相关的一个元信息就是该 inode 的引用计数。系统如果发现某个 inode 的引用计数变为0，就会删除这个 inode 及其对应的 data block。而我们常用的 `rm` 命令，其实叫做 `unlink` 更好，rm 命令的作用仅仅是删除了 directory entry 中 i-node number  和 filename 的关联关系而已，此时 inode 的引用计数会减一，如果减一后引用计数的值恰好是 0 ，便会触发删除操作。

而对于本文开头的例子，执行完 `rm /tmp/mysql_query.log` 后，文件 /tmp/mysql_query.log 的引用计数减一，但是减一后是否为零呢？必然不是，因为 msyql 进程正打开这个文件呢，就会造成 /tmp/mysql_query.log 成为 phantom file ，即： **通过系统文件树是看不到了，比如 ls,du 命令，但是该文件对应的 inode 结点和 data block 还在，并没有释放硬盘空间** 。只有当 mysql 进程关闭或者重启，关闭该文件句柄，对 /tmp/mysql_query.log 对应的 inode 结点引用计数再减一变为零，才会引发删除操作，硬盘空间得以释放。

#### 不重启守护进程的解决方法

仔华以前常推荐清理测试环境 mysql_query.log 的方法，就是 `cat > /tmp/mysql_query.log` 。这样就可以达到在不重启 mysql 的情况下，释放硬盘空间，还不影响 mysql 的查询日志继续 APPEND 。为何这个命令这么神奇？我们一步步来剖析。

首先，bash 的标准输出重定向一个文件是如何来做呢？

[Advanced Bash-Scripting Guide](http://www.tldp.org/LDP/abs/html/) 的 [Chapter 20. I/O Redirection](http://www.tldp.org/LDP/abs/html/io-redirection.html) 中的说明

~~~ shell
> filename    
  # The > truncates file "filename" to zero length.
  # If file not present, creates zero-length file (same effect as 'touch').
  # (Same result as ": >", above, but this does not work with some shells.)
~~~
      
以及 [bash 4.3](https://ftp.gnu.org/pub/gnu/bash/bash-4.3.tar.gz) 源码 make_cmd.c L700 中 

~~~ C
switch (instruction)
  {

  case r_output_direction:        /* >foo */
  case r_output_force:        /* >| foo */
  case r_err_and_out:         /* &>filename */
    temp->flags = O_TRUNC | O_WRONLY | O_CREAT;
    break;
~~~

是通过 `open(filename, O_TRUNC | O_WRONLY | O_CREA)` ，其中 O_TRUNC 直接将文件给 truncate 成 zero length 了。具体的做法我猜测是：直接修改该文件对应的 inode ，去除所有指向 data block 的指针。（具体需要看 Linux 源码核实，我没有查到资料。）

而这样的做法是否影响 mysql 进程向日志 APPEND 查询日志呢？

![Kernel data structures for open files](/images/open-files.gif)

此图摘自 《Advanced Programming in the UNIX Environment》3.10 File Sharing 。细节我就不解释啦。

对于 mysql 进程 APPEND 查询日志，使用 `open(filename, O_APPEND | O_WRONLY | O_CREAT)` 打开日志。

* file table 中的 file status flags 肯定是 O_APPEND ，APPEND 日志的时候，都是 lseek 到文件尾部进行 write，并且还保证原子性。
* 上面的猜测， **The > truncates file "filename" to zero length.** 仅仅更新的是 inode 。

如果要想直接将文件给 truncate 成 zero length ，也可以使用 [truncate](http://linux.die.net/man/2/truncate) 。所以 `cat > /tmp/mysql_query.log` 是个好方法，这种方法具有普适性。

比较好的开源程序中，都考虑了这一点，删除日志就简单很多，在 rm 后直接 **reopen file** ，比如:

* nginx 的 `kill -s USR1 <pid>`
* mysql 的 `flush logs;`


#### 延伸阅读

大家看看这个帖子，有点意思。 [[OT]python写的 下厨房 被黑了。](https://groups.google.com/forum/#!msg/python-cn/i--6-0giwUk/n_RNDSzCv70J) ，后面有讨论如果 mysql 数据库文件仅仅是被 rm 了，mysql 还在运行，可以恢复吗？


#### 参考

* 《Advanced Programming in the UNIX Environment》
* [Advanced Bash-Scripting Guide](http://www.tldp.org/LDP/abs/html/)
