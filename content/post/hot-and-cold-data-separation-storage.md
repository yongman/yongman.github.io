---
title: 对数据冷热分离存储的思考
categories: [存储]
date: 2018-09-06 23:54:21
---

对于冷热数据分层存储的最直接的目的就是节省成本，计算机结构里，内存->nvme ssd->ssd->机械盘，访问速度依次降低，单位成本依次降低，存储密度依次增大。
![](https://raw.githubusercontent.com/yongman/i/img/picgo/CFkzyQJ.png)

对于像`redis`这种天生为高速大并发设计的高性能系统，数据存储也理应放在内存。

但是我们大多数的使用`redis`的场景可能并不是所有数据冷热度是相同的，有些时候我们的系统中也实在用不到100%的redis性能，能满足场景需求的前提下，节省成本就成了开发者或者云平台要考虑的事情了。

冷热数据分层存储概念也很容易理解，因为计算机系统中到处都是分层的例子，内存的出现也是因为磁盘速度太慢了，cpu如果直接在磁盘上操作，那一个命令估计要等上几秒钟了，cpu中还有L1 cache和L2 cache，都是为了不同组件之间的速度的匹配。

对于`redis`的应用场景来说，完全可以通过一定的算法，对访问进行统计，将冷数据落盘存储，在访问内存中的索引，发生了`缺页中断`时，需要将持久化的数据再load到内存。这样实现的是对内存的扩充，而不是对磁盘的cache。
![](https://raw.githubusercontent.com/yongman/i/img/picgo/sC1Fwjj.png)
都是高速+低速设备，两者的不同在：

- 对内存的扩充：所有的数据读写都在内存，不需要每次写入都更新磁盘，而是在数据按一定策略需要置换出去的时候才进行落盘，所以大部分场景，性能影响不大。
- 内存用作cache的场景：读写都先请求cache，读不到就访问磁盘，然后回填内存，写入的话必须要保证每次都写入磁盘，适合读多写少的场景。

对于`redis`的冷热分离，redislabs在几年前就推出了`Redis On Flash`，阿里云也针对`Redis`做了类似的混合存储，实现冷热分离，无一例外，他们都使用了`RocksDB`用作磁盘的存储。可以在保持成本低和性能差距不大的前提下，获得更大存储容量。
![](https://raw.githubusercontent.com/yongman/i/img/picgo/AUbxmS9.png)

对于`缺页中断`的处理肯定会影响连接的吞吐，这个需要努力做的是尽量减少`缺页中断`处理的时间，可以采用nvme ssd新硬件，实现用户态IO来避免用户态和内核态切换开销。

当然难点还是有的，在完全兼容社区redis的前提下，对于单节点存储上百G的数据，对于故障处理或者网络中断等情况的数据恢复如何高效实现等、`缺页中断`处理如何能使用户无感知、对于集合类的大key如何实现高效的冷热存储等。

**参考**
1. [Running Out of RAM on Redis? No More OOM With Hybrid Memory (RAM and Flash) - DZone Performance](https://dzone.com/articles/hybrid-memory-using-ram-amp-flash-in-redis)
2. [Redis on Flash Overview - Redis Enterprise Software | Redis Labs](https://redislabs.com/redis-enterprise-documentation/concepts-architecture/memory-architecture/redis-flash/)
3. [云数据库Redis混合型存储冷热数据分离节约成本](https://promotion.aliyun.com/ntms/act/hybridstore.html)
4. [微信 PaxosStore:海量数据冷热分级架构](https://cloud.tencent.com/developer/article/1020347)
