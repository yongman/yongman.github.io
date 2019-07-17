---
title: LevelDB版本管理和读写操作
categories: [存储]
date: 2018-08-01 20:07:32
---

LevelDB是一个轻量级的key value存储系统，存储结构采用LSM-Tree，对写操作优化，特别是普通的机械盘。

LevelDB是Bigtable中描述的的key value存储系统的实现，它具有的一些特性：

- 二进制安全的kv存储
- 数据存储方式按key排序
- 支持批量原子操作
- MVCC和快照
- 数据遍历
- 支持数据压缩
- 多线程安全

### LSM和B+Tree

一张图描述B树
![B树](https://ws4.sinaimg.cn/large/006tKfTcgy1ftt3tnl3m3j30hs07e3yq.jpg)
B树主要用于文件系统

一张图描述B+树
![B+树](https://ws2.sinaimg.cn/large/006tKfTcgy1ftt40kjr9tj30hs072t8z.jpg)
常见的磁盘存储引擎通常采用B+树进行索引，B+树是专门对磁盘设计的一个N叉的排序树，数据查找过程中索引的目的就是加快查找速度，减少磁盘IO，一般采用B+树的存储引擎，父节点中的元素都会出现的子节点，叶子节点包含了数据全量元素信息，叶子节点形成了一个有序链表。

数据存储在B树和B+树的区别
![B树](https://ws1.sinaimg.cn/large/006tKfTcgy1ftt46ur0p8j30hs08mt96.jpg)
![B+树](https://ws2.sinaimg.cn/large/006tKfTcgy1ftt47thrclj30hs0850t5.jpg)

由于B+树的中间节点只存储索引，所以单个节点存储更多的元素，所以同样的数据量，B+树IO次数更少。

LSM树是一个N阶合并树，数据写入先追加日志，然后更新内存，内存达到一定阈值后dump到磁盘，并周期性进行compaction。由于LSM的特性，将随机写转化为顺序写，防止随机磁盘IO。

由于LSM的特性和LevelDB的分层，其中的数据经过多次compaction后冷热已经实现了分离(按最后一次更新时间)，由于memtable的存在，看似像是缓存，但行为和缓存是不一样的，普通的缓存的设计思想是当从缓存读取失败后进行填充和主动更新数据后主动更新缓存，它是把经常访问的数据放到离业务更近的位置；而LevelDB是对于新写入的数据会放在memtable，当memtable刷到磁盘后，再读取相关的数据，数据并不会回填到内存，即便一直在读取的热数据，当没有更新时，LevelDB会尽量将数据放到离业务更远的位置。

### Version

Version就是LevelDB中的所有数据的一个版本或者视图的元信息，其中记录了每一层的所有文件和一些compaction相关状态信息。

VersionSet是用来管理Version的一个集合。
```
VersionSet=Version1+Version2+Version3+...+currentVersion
```
![VersionSet](https://ws4.sinaimg.cn/large/006tKfTcly1fttzvz1nfdj30pe0fpabv.jpg)

VersionEdit可以看做是Version之间变更的增量信息，可以从上一个版本与VersionEdit合并，生成新的版本。在开始启动时，LevelDB会读取Manifest文件，按顺序读取VersionEdit，组成VersionSet，通过Builder生成currentVersion
```
Vn=Vn-1+VersionEdit
```
![VersionEdit](https://ws4.sinaimg.cn/large/006tKfTcgy1fttzyavx74j30gd04hjrh.jpg)

__Write/Put/Delete操作__

Write操作流程很简单，但是看LevelDB的代码，感觉写的很奇妙，其实就是一个生产消费者模型，它的实现的区别在生产消费者角色不是固定的，一个生产者线程，后续的操作可能被选择为消费者。当有多个生产者线程写入数据，数据被封装成Writer加入队列，然后判断是否被处理过或者是不是在队列头的第一个任务，如果是处在队列头的第一个任务，那这个Writer的角色转化成消费者，批量将队列任务进行处理，然后标记完成处理过的任务，当再次唤醒对应的生产者线程，任务已经被处理过，直接返回。

* 实现简洁，不需要区分生产消费者，生产者就是消费者；
* 消费者批量处理生产者请求；
* 可以保证同一时刻只有一个消费者，尽量缩小临界区，提升并发度。

__Get操作__

整个Get流程也很清晰，先从memtable中查找，找不到就再次查找immutable memtable中查找，如果immutable memtable中也不存在，最后会在currentVersion中查找，整个查找过程并不是直接查找user key，而是通过user key和sequence number生成的lookup key，增量序列号来支持快照读。

最后就是更新统计信息，检查是否需要触发compaction。


### 参考内容
1. [CiteSeerX — The Design and Implementation of a Log-Structured File System](http://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.41.8933)
2. [庖丁解LevelDB之版本控制 | CatKang的博客](http://catkang.github.io/2017/02/03/leveldb-version.html)
