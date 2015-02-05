---
layout: post
title: "为什么我们不使用Redis做存储数据库？"
date: 2015-02-05 09:59
comments: true
categories: 
- Redis
---

[Redis](http://redis.io/) 以其强劲的性能、完善的数据结构、丰富的数据操作，再加上数据持久化、数据同步，以碾压 memcached 的姿态迅速在我们的项目中普及。但是在我们大部分的应用，Redis 承担的角色都是缓存数据库，要么后端还有其他数据库做永久存储使用，要么存储的数据是临时的，允许一定程度的数据丢失。

这次我们基于 Cotton 在上层搭建一个类 filepicker 的服务，需要在数据库中存储文件的元信息。基于 Cotton 的经验，400万文件占用的所有 Redis 内存是1.5G，还不说这个400万个文件都有冗余备份。基于这样的场景类比，觉得使用 Redis 作为存储数据库，内存占用不是什么问题。再加上 Redis 其他数据库无法比拟的性能，想想都醉了。所以就开始了使用 Redis 做存储数据库的思考。

细想之下要想 Redis 作为存储数据库， Redis 需要满足3点：

1. 数据 **实时** 持久化。
2. 副本集。
3. 扩容。

#### 数据实时持久化

Redis 提供了两种 [持久化存储](http://redis.io/topics/persistence) 的方式：

* RDB 持久化。可以在指定的时间间隔内生成数据集的时间点快照。优点是：文件紧凑，占用空间小，用 RDB 文件恢复速度快，适合作为备份及灾难恢复。但是单次 RDB 开销比较大，只能设置间隔一段时间执行。如果两次 RDB 的间隔，Redis 挂掉，这间隔的数据更新就丢失了。
* AOF 持久化。记录服务器执行的所有写操作命令，并在服务器启动时，通过重新执行这些命令来还原数据集。 AOF 文件中的命令全部以 Redis 协议的格式来保存，新命令会被追加到文件的末尾。 Redis 还可以在后台对 AOF 文件进行重写（rewrite），使得 AOF 文件的体积不会超出保存数据集状态所需的实际大小。因此，AOF 文件必然会比 RDB 文件占用的空间大，恢复数据时挨个执行一遍写命令，恢复速度自然比不上 RDB，在开启 `fsync` 的情况下，速度会比 RDB 慢。但是最大的优点是：可以 **准实时** 持久化，通过配置 `fsync` 策略，最大可能减少数据丢失的风险。

  - fsync 配置的策略是每秒 append 下 Redis 写命令，最多只会丢失这一秒的数据。Redis 默认的 fsync 策略。
  - fsync 配置的策略是同步 append Redis 写命令，这样不会丢数据，但是性能就没有那么快了。

聪明的你肯定会发现，能不能这样干呢：	同时使用 AOF 和 RDB 持久化，平时 append 写命令到文件，定时 RDB 持久化同时清空 AOF 文件，从 RDB 持久化后的时间点重新 AOF，恢复的时候先从 RDB 恢复再从 AOF 恢复。看起来不错哦，只是目前 Redis 不支持，让我们憧憬下美好的未来。

> Note: for all these reasons we'll likely end up unifying AOF and RDB into a single persistence model in the future (long term plan).
    
在目前阶段，使用 AOF 也不错。

* 占用空间大点，不就是耗费点磁盘空间。
* 恢复速度慢点，其实不算慢。顾爷在公司分配的虚拟机上，400万写命令也就21秒。
* 如果 fsync 同步写，确实慢很多，但是改成每秒，也能接受。再说 filepicker 典型的写多读少，即使开启 fsync 同步写，也不为过。

#### 副本集

前面讲到，使用 AOF 也不错，如果能包容 AOF 的缺点，AOF 能够提高实时持久化，其 AOF 文件也能拿来定时备份，以及容灾恢复，貌似很不错的选择。但是定时备份依然有丢数据的风险，比如定时备份的间隙磁盘挂了，不能读盘了。因此最好的方法就是 Replica Sets ，保存副本集，既能够实时备份，又能够快速容灾恢复。不用说，实现 Replica Sets 的方式，就是 Master-Slave 架构了。

	+---------+          +---------+
	|  Master |  <=====  |  Slave  |
	+---------+          +---------+

那么问题来了？使用 Master-Slave 架构后，Master-Slave 是否需要开启 AOF 持久化？有一种观点是： **Master 不开持久化保证访问性能，Slave 开 AOF 持久化（哪怕 fsync alwarys 都可）保证数据不丢失** 。貌似看起来 perfect 的方案！！有木有风险？

1. Master 如果重启会发生什么？Master 重启后，由于没有开启持久化，不需要从 AOF 文件恢复数据，重启后，会是一个空的数据集，此时 Slave 监测到了 Master 起来了重新同步，从而也得到 **一个空的数据集** 。你没有看错，数据因此就丢了。因此如果 Master 没有开启持久化，那么 Master 就一定不能随便重启，包括随便自动重启、开机启动等。
2. 如果保证 Master 不会自动重启、随机重启后，还有木有问题呢？我们想想这样一种场景，Master 挂了，也没有随机重启，Slave 上面的数据还没有丢，此时为了恢复服务，需要将 Slave 变成 Master，原来的 Master 恢复后变成 Slave 从新 Master 同步数据，角色互变后，此时的 Master 开启了持久化，Slave 关闭了持久化。为了维护我们初衷，需要手动关闭 Master 的持久化以及自动重启，手动开启 Slave 的持久化和自动重启。这对于运维来说，是个灾难。

因此从这里来说，还是 ** Master-Slave 都要开启持久化比较好** 。那如果 Master 挂了后，该如何处理呢？是否应该将 Slave 变成 Master 去承载服务呢？这样就涉及到了一个问题，Master 和 Slave 的数据谁更新？

* 如果 Master 开启了 `fsync always` ，由于 Slave 同步延迟的缘故，Master 的数据会更新。
* 如果 Master 开启了 `fsync every second` ，这个时候 **有可能** 数据已经同步给了 Slave ，但是还没有 fsync 到 Master 的 AOF 文件，所以有可能 Slave 数据更新。

头痛啊，到底哪个数据更新，该从哪个恢复呢？又给运维带给了困难。

#### 扩容

由于 Redis 的数据都必须在内存中，内存资源有限，随着数据量上涨，单台服务器必然不能承载，因此需要横向扩容。很多童鞋觉得，Redis 扩容很方便，只要根据 KEY 进行哈希算法（比如一致性哈希）即可分布式。使用 Redis 作为数据缓存确实可以这种，当对 KEY 进行哈希算法寻找 Shard 节点，扩容后如果查询没有命中，从后端的数据存储数据库取数据，重放数据到该 Shard Redis 节点中。但是结合我们今天的主题，Redis 就是作为后端的数据存储数据库哦，扩容 Rebalance 的过程如何实现？或许有 Rebalance 的方法，但是是否成熟，是否有坑儿，项目是否敢试用？

#### 结论

使用 Redis 作为存储数据库，我的结论是：

- 数据库持久化 [Persistence](http://redis.io/topics/persistence) 的方法比较成熟。
- 副本集 [Replication](http://redis.io/topics/replication) 的运维有一定难度。
- 扩容 Rebalance 还不见官方方案。

所以要慎用哦。
