---
title: Percolator论文学习
date: 2018-06-22 14:44:15
categories: [分布式]
toc: true
---

Percolator是Google为了解决搜索引擎中增量索引，替代原先的MR系统，可以实现增量更新索引，使得新的网页更快的被用户搜索到。

离线的处理方式可以很好的实现索引的相关计算，MR吞吐量在一定周期内也可以完成整个索引库的更新，但是对于只重新抓取了一小部分页面时，对于MR来说效率很低，为了少量的增量更新能够被索引，全量数据需要全部计算一遍。

理想的情况就是索引可以被流式增量处理，任何新的更新可以在一个大型的文档库中实时进行小的更新。

Percolator提供在PB级别存储库中随机访问的能力。随机访问允许我们单独的处理文档，避免全局的扫描(未优化的MR往往需要全局扫描)。为了提升吞吐量，percolator实现高并发事务。

论文中主要介绍了分布式事务的实现和通知机制，这里注重关注分布式事务。

### 分布式事务

Percolator运行于BigTable之上，支持快照隔离，采用MVCC维护数据多版本。在分布式系统中，要保证事务的正确性，需要有一个严格递增的版本号，percolator中使用timestamp oracle来生成全局严格递增的时间戳，每个事务需要获取两次时间戳，第一次是事务开始时，获取事务开始时间戳，第二次是事务提交时，用作提交时间戳。

对于任何一列数据C，在bigtable中存储格式会分为：
* C:data 用于存储真正的数据
* C:lock 用于存储事务的锁信息
* C:write 用于存储数据的提交版本

以论文中转账的例子来进行说明
![1.jpg](http://ww1.sinaimg.cn/large/6d6b007fly1fsoph426dij20gs0cujt4.jpg)
![2.jpg](http://ww1.sinaimg.cn/large/6d6b007fly1fsoph44m3hj20ge0j877u.jpg)
![3.jpg](http://ww1.sinaimg.cn/large/6d6b007fly1fsoph43k62j20hl0b9wg5.jpg)
![4.jpg](http://ww1.sinaimg.cn/large/6d6b007fly1fsoph45dc9j20ie0ljdjy.jpg)

事务分两个阶段，prepare阶段和commit阶段。

* Prepare阶段：

1. 启动事务T1，获取时间戳7
2. 选择Bob:lock为primary lock，写入lock列，然后将新的数据写入C:data列
3. 事务使用同一个时间戳写入Joe:lock列，值为事务的primary key，指向Bob,并将新的数据写入Joe:data列

* Commit阶段：

1. 准备提交，首先获取时间戳8
2. 清理Bob的lock列，以新时间戳写入一条数据8， 向Bob write列以时间戳8写入一条数据，数据中包含指向最新的数据指针，指向事务开始时间戳7中对应的数据
3. 清理Joe中的lock列，以新时间戳写入一条数据8，向Joe write列写入一条数据，指向事务开始时间戳中对应的数据


### 事务冲突

* 写写冲突

事务开始时间戳之后，write列有更新的时间戳记录，说明已经有事务在本事务开始之后还未提交之前已经提交，发生冲突。或者lock列中存在锁，也直接取消。

* 读写冲突

Get操作会先进行检查(0,开始时间戳)检查锁，如果存在锁，说明前面的事务还未完成，读事务等待锁释放。

* 锁清理

事务处理过程中，prewrite阶段，客户端故障，会导致事务锁残留在系统中，采用lazy方式来进行锁清理，当事务A发现锁，并能确定该锁被事务B遗弃，事务A会将锁进行清理。

commit阶段客户端故障，如果已经提交过一个写记录，并且还有未提交的记录，执行roll-forward，primary已经被替换为一条写记录，事务必须被提交，否则应该roll-back。

