---
layout: post
title: "MongoDB磁盘占用"
date: 2015-03-10 18:50
comments: true
categories: 
---

在 UU 打算将 MongoDB 迁移至 Ocean 的时候，熊雄问我，目前 UU MongoDB 占用多少磁盘空间，500GB 够么？因为一个 Ocean 高性能节点的 SSD 存储空间是 500GB 。此时该如何 **定位数据库真实占用的磁盘空间，并且根据产品的发展对将来的磁盘占用增长做一个合理的预估** ？恩，这篇文章就来好好探讨下这个问题。

一般第一直觉就是通过 Linux `du` 系统命令查看 MongoDB [数据目录](http://docs.mongodb.org/manual/reference/glossary/#term-data-directory) 的大小，以此衡量 MongoDB 大概的磁盘占用。这样得到的结果也不能真实反映 MongoDB 数据占用的真实物理空间。因为：

1. MongoDB 会预分配指定大小数据文件(data file)，防止磁盘碎片。比如为数据库 `my-db` 分配的第一个数据库文件 `my-db.0` 为 64MB，第二个数据文件 `my-db.1` 为 128MB，这样指数增长，一直到 2GB。由于是预分配文件，必然有文件空间没有使用，最多有可能是 2GB 的数据文件空间没有利用上。
2. oplog。在 MongoDB replica set，包含 oplog.rs 文件，也会预分配一定的空间占用。在64位机器上的 MongoDB 默认会预分配 5% 的磁盘空间。
3. journal 。在没有落地到磁盘的写操作日志会占用一定的磁盘空间。
4. 空记录。对于已经删除的文档和表空间，MongoDB 并不会将这些磁盘空间交还给操作系统，而是依然占着，留着以后用。

由于这些额外的空间占用，使得我们通过 du 命令获取文件或者目录大小来衡量数据库的磁盘占用就不是那么精准了。MongoDB 也意识到了这个问题，提供了 [dbStats](http://docs.mongodb.org/manual/reference/command/dbStats/) 命令来精确查询数据库的磁盘空间占用。但是这个命令一下子抛出 `dataSize` ， `storageSize` ，`indexSize` ， `fileSize` ，官方文档的解释也比较深奥，不容易看懂哇。

为了搞懂这些 `*Size` 的具体含义，需要搞懂 MongoDB 到底是如何存储数据的？前面提到 MongoDB 通过数据文件存储数据，存储数据库 `my-db` 的数据文件分别是 `my-db.0` ， `my-db.1` .. `my-db.N` 。每个数据文件又很多 `extents` 组成，这些 `extents` 可以存储文档数据、索引以及 MongoDB 生成的一些元信息。

![MongoDB extents](http://blog.mongolab.com/wp-content/uploads/2014/01/data_extents1.png)

* 每一个 extents 都只属于一个 Collection，只能保存这个 Collection 的文档和索引。
* 每一个 extents 或者保存文档，或者保存索引，不能同时保存文档和索引。
* 每一个 Collection 有很多个 extents 组成。
* 当需要创建一个 extents，会在数据文件中申请新的空间创建新的 extents ，如果数据文件空间不够，就创建新的数据文件。

了解完 extents 后，再来挨个阐述 `dataSize` ， `storageSize` ，`indexSize` ， `fileSize` 就方便很多。

#### dataSize

![MongoDB dataSize](http://blog.mongolab.com/wp-content/uploads/2014/01/data_size-1024x385.png)

dataSize 就是由这些文档占用的空间累加所得（包含 padding 哦）。

* 当文档被删除时，dataSize 也随着变小。
* 但是当文档被 shrink 时，dataSize 并不会变小，因为文档依然占据这原来分配的空间。
* 当文档被修改时，如果原有的空间（包含 padding）够用，不会分配新的文档空间啦。

#### storageSize

![MongoDB storageSize](http://blog.mongolab.com/wp-content/uploads/2014/01/storage_size-1024x407.png)

storageSize 就是 dataSize 加上被删除文档（空记录）占用的空间（前面提到，删除文件的空间 MongoDB 依然占着，不会交还给操作系统的）。因为当文档被删除时，storageSize 也不会变小。

#### fileSize

![MongoDB fileSize](http://blog.mongolab.com/wp-content/uploads/2014/01/file_size-1024x430.png)

fileSize 就是所有文档空间占用，索引空间占用，预分配（还未使用）空间占用之和。fileSize 和前面提到通过 du 系统命令查看数据文件的磁盘占用能基本吻合。fileSize 自然大于 storgeSize。当删除文档、表空间时，fileSize 都不会变小，因为 MongoDB 并没有把释放的空间交还给操作系统。只有当删除整个数据库时，fileSize 才会减少。

回到我们开头说的问题，如果将 UU MongoDB 迁移到 Ocean，占用的初始空间有多少呢？大概是

    dataSize + indexSize + data file预分配空间（不会超过2GB） + oplog + journal

因为刚刚迁移时，文档中不会存在太多被删除的文档空间。

#### References && Resources

* [Why are the files in my data directory larger than the data in my database?](http://docs.mongodb.org/manual/faq/storage/#why-are-the-files-in-my-data-directory-larger-than-the-data-in-my-database)
* [How big is your MongoDB?](http://blog.mongolab.com/2014/01/how-big-is-your-mongodb/)
