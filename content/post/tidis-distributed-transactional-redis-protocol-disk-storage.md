---
title: Tidis - 基于tikv实现兼容redis协议分布式事务存储
date: 2018-07-17 09:40:27
categories: [分布式]
---

## 1. 背景

redis本身的定位是高速缓存，有多种数据结构支持，全量内存，使用成本高，当数据量达到一定规模后，用户不得不考虑将部分冷数据从redis中迁移到其他存储，节省成本。也有很多应用场景，用户使用redis的目的很大一部分是由于redis api使用简单，数据结构丰富，快速实现业务功能，而真正线上服务的qps远远达不到需要使用redis的场景，造成资源浪费严重。

## 2. 相似项目调研

redis定位高性能高速缓存，低容量，低延时，当容量达到一定规模，成本可能成为限制数据规模的因素。当用户的应用场景是想使用redis的数据结构，作为可靠的存储、对数据安全性要求高、数据量大，数据存储介质就需要从昂贵的ram存储到磁盘或ssd。


1. 基于单机key-value引擎，增加数据结构和协议支持

* 1.1 **SSDB**
![SSDB架构图](https://camo.githubusercontent.com/6f3243b32deae6f762859f3ee91eeeb36f291518/687474703a2f2f737364622e696f2f737364622e706e67)
  - [repo](https://github.com/ideawu/ssdb)
  - 底层存储引擎采用leveldb单机引擎，实现zset、map、queue数据结构
  - 基于master-slave复制、
  - 全量复制和flush操作速度慢
  - 本身无集群化方案，类似于redis2.8
  - 主从复制效率低，全局锁竞争严重
  - 数据结构的一些操作效率低，很多操作基于遍历
  - c++实现


* 1.2 **LedisDB**
实现架构类似ssdb
  - [repo](https://github.com/siddontang/ledisdb)
  - golang实现
  - 底层引擎采用可替换引擎，leveldb、rocksdb、goleveldb等，实现kv、list、set、zset、hash等数据结构
  - 采用leveldb或rocksdb会存在cgo性能损失
  - 基于master-slave复制
  - 采用类似codis的集群化方案xcodis
  - 未经线上多规模验证


* 1.3 **RedisDB**
  - [repo](https://github.com/ksarch-saas/ssdb)
  - 基于SSDB二次开发(c++)
  - 优化主从复制
  - 增加数据多版本
  - 支持所有数据自动过期
  - 支持数据迁移
  - 采用修改过的codis集群化方案(golang)
  - 未大规模线上验证


* 1.4 **swapdb**
![swapdb](https://github.com/JingchengLi/swapdb/raw/master/docs/fundamental.jpg)
  - [repo](https://github.com/JingchengLi/swapdb)
  - 深度定制的redis和优化过的ssdb整合
  - 京东金融开源
  - 高度兼容redis api
  - 冷热数据分离，热数据在redis，冷数据存储到ssdb
  - 可以实现原生的redis cluster集群
  - 无数据可靠性保证，定位还是缓存
  - 开源后没看到社区应用案例

* 1.5 **Tidis**
![tidis架构图](https://github.com/yongman/tidis/raw/master/docs/tidis-arch.png)
  - [repo](https://github.com/yongman/tidis)，欢迎star
  - golang实现
  - 目前状态为demo实现，还处于功能开发阶段，仅供测试
  - 实现采用分布式key value引擎TiKV， 可以对比FoundationDB
  - Tidis类似TiDB层，无状态
  - 采用存储计算分离架构
  - 无限水平扩展
  - 自动数据均衡
  - Raft副本复制，确保数据不丢
  - TiKV采用rust语言，底层引擎为rocksdb
  - 支持分布式事务
  - 事务采用乐观锁，事务提交冲突可配重试次数


## 3. 对比

上面列出的5种存储方案中SSDB/ledisdb/redisdb是同一类型，基于单机的kv引擎实现的类redis协议的存储，数据可靠性采用异步副本复制，集群化都需要额外的工作，集群化后需要实现数据动态迁移功能，如果要保证数据安全性，需要采用raft等同步协议，复杂度较高。
* 优势：属于传统实现方式，实现单机存储引擎，性能好，大部分取决于磁盘、底层使用的存储引擎和数据组织方式，网络性能损耗小。
* 劣势：
  - 不支持事务，自己实现集群化方案和数据复制和迁移，复杂度高。
  - 采用悲观锁，适合有很多热点写入的数据场景


swapdb的实现方式很美好，由存储端完成冷热数据的区分，不需要用户关心，用户只需要像使用redis一样来访问，冷数据会落盘，不占用内存，达到节省成本的目的。swapdb虽然支持原生的cluster方式，把内存数据和磁盘数据来一起运维，两种数据是一个整体，原来的cluster特性会由于ssdb数据部分操作缓慢导致运维上的不可控等因素。
* 优势：屏蔽用户使用场景，冷热数据自动分离存储。
* 劣势：快速和慢速数据同时存放，运维难度大

Tidis采用计算存储分离设计，底层采用分布式kv存储，分布式事务实现方式为2pc，所有的写入操作都需要起事务，事务时间戳从PD节点获取，同时对相同的key进行写入，事务提交会产生冲突导致事务重试，并且事务都是2pc，两次网络交互会增加请求latency，优势是极大简化了计算层的逻辑，存储层实现了分布式事务，实现了raft数据复制，实现了数据动态迁移和节点高可用切换，而计算层需要做的就是专注于数据结构实现效率和异常处理等逻辑。
* 优势：存储计算分离，存储引擎保证了分布式事务，实现了数据复制和自动均衡，计算层不需要实现分布式的逻辑。
* 劣势：
  - latency比较高，一次写入可能需要三次网络交互，1次从pd获取时间戳，2次阶段提交
  - 采用事务乐观锁，正常的写入不会进行加锁，commit时检测事务是否存在冲突，进行重试，不适合很多热点数据写入的数据场景
  - 写密集型场景容易发生写冲突，导致事务commit失败重试

![tikv](https://raw.githubusercontent.com/pingcap/tikv/master/images/tikv_stack.png)

## 4. 实现
**已经实现的命令**

### 4.1 string

    +-----------+----------------------------------+
    |  command  |              format              |
    +-----------+----------------------------------+
    |    get    | get key                          |
    +-----------+----------------------------------+
    |    set    | set key value                    |
    +-----------+----------------------------------+
    |    del    | del key1 key2 ...                |
    +-----------+----------------------------------+
    |    mget   | mget key1 key2 ...               |
    +-----------+----------------------------------+
    |    mset   | mset key1 value1 key2 value2 ... |
    +-----------+----------------------------------+
    |    incr   | incr key                         |
    +-----------+----------------------------------+
    |   incrby  | incr key step                    |
    +-----------+----------------------------------+
    |    decr   | decr key                         |
    +-----------+----------------------------------+
    |   decrby  | decrby key step                  |
    +-----------+----------------------------------+
    |   strlen  | strlen key                       |
    +-----------+----------------------------------+
    |  pexpire  | pexpire key int                  |
    +-----------+----------------------------------+
    | pexpireat | pexpireat key timestamp(ms)      |
    +-----------+----------------------------------+
    |   expire  | expire key int                   |
    +-----------+----------------------------------+
    |  expireat | expireat key timestamp(s)        |
    +-----------+----------------------------------+
    |    pttl   | pttl key                         |
    +-----------+----------------------------------+
    |    ttl    | ttl key                          |
    +-----------+----------------------------------+

### 4.2 hash

    +------------+------------------------------------------+
    |  Commands  | Format                                   |
    +------------+------------------------------------------+
    |    hget    | hget key field                           |
    +------------+------------------------------------------+
    |   hstrlen  | hstrlen key                              |
    +------------+------------------------------------------+
    |   hexists  | hexists key                              |
    +------------+------------------------------------------+
    |    hlen    | hlen key                                 |
    +------------+------------------------------------------+
    |    hmget   | hmget key field1 field2 field3...        |
    +------------+------------------------------------------+
    |    hdel    | hdel key field1 field2 field3...         |
    +------------+------------------------------------------+
    |    hset    | hset key field value                     |
    +------------+------------------------------------------+
    |   hsetnx   | hsetnx key field value                   |
    +------------+------------------------------------------+
    |    hmset   | hmset key field1 value1 field2 value2... |
    +------------+------------------------------------------+
    |    hkeys   | hkeys key                                |
    +------------+------------------------------------------+
    |    hvals   | hvals key                                |
    +------------+------------------------------------------+
    |   hgetall  | hgetall key                              |
    +------------+------------------------------------------+
    |   hclear   | hclear key                               |
    +------------+------------------------------------------+
    |  hpexpire  | hpexpire key int                         |
    +------------+------------------------------------------+
    | hpexpireat | hpexpireat key ts                        |
    +------------+------------------------------------------+
    |   hexpire  | hexpire key int                          |
    +------------+------------------------------------------+
    |  hexpireat | hexpireat key ts                         |
    +------------+------------------------------------------+
    |    hpttl   | hpttl key                                |
    +------------+------------------------------------------+
    |    httl    | httl key                                 |
    +------------+------------------------------------------+

### 4.3 list

    +------------+-----------------------+
    |  commands  |         format        |
    +------------+-----------------------+
    |    lpop    | lpop key              |
    +------------+-----------------------+
    |    rpush   | rpush key             |
    +------------+-----------------------+
    |    rpop    | rpop key              |
    +------------+-----------------------+
    |    llen    | llen key              |
    +------------+-----------------------+
    |   lindex   | lindex key index      |
    +------------+-----------------------+
    |   lrange   | lrange key start stop |
    +------------+-----------------------+
    |    lset    | lset key index value  |
    +------------+-----------------------+
    |    ltrim   | ltrim key start stop  |
    +------------+-----------------------+
    |    ldel    | ldel key              |
    +------------+-----------------------+
    |  lpexipre  | lpexpire key int      |
    +------------+-----------------------+
    | lpexipreat | lpexpireat key ts     |
    +------------+-----------------------+
    |   lexpire  | lexpire key int       |
    +------------+-----------------------+
    |  lexpireat | lexpireat key ts      |
    +------------+-----------------------+
    |    lpttl   | lpttl key             |
    +------------+-----------------------+
    |    lttl    | lttl key              |
    +------------+-----------------------+

### 4.4 set

    +-------------+--------------------------------+
    |   commands  |             format             |
    +-------------+--------------------------------+
    |     sadd    | sadd key member1 [member2 ...] |
    +-------------+--------------------------------+
    |    scard    | scard key                      |
    +-------------+--------------------------------+
    |  sismember  | sismember key member           |
    +-------------+--------------------------------+
    |   smembers  | smembers key                   |
    +-------------+--------------------------------+
    |     srem    | srem key member                |
    +-------------+--------------------------------+
    |    sdiff    | sdiff key1 key2                |
    +-------------+--------------------------------+
    |    sunion   | sunion key1 key2               |
    +-------------+--------------------------------+
    |    sinter   | sinter key1 key2               |
    +-------------+--------------------------------+
    |  sdiffstore | sdiffstore key1 key2 key3      |
    +-------------+--------------------------------+
    | sunionstore | sunionstore key1 key2 key3     |
    +-------------+--------------------------------+
    | sinterstore | sinterstore key1 key2 key3     |
    +-------------+--------------------------------+
    |    sclear   | sclear key                     |
    +-------------+--------------------------------+
    |   spexpire  | spexpire key int               |
    +-------------+--------------------------------+
    |  spexpireat | spexpireat key ts              |
    +-------------+--------------------------------+
    |   sexpire   | sexpire key int                |
    +-------------+--------------------------------+
    |  sexpireat  | sexpireat key ts               |
    +-------------+--------------------------------+
    |    spttl    | spttl key                      |
    +-------------+--------------------------------+
    |     sttl    | sttl key                       |
    +-------------+--------------------------------+

### 4.5 sorted set

    +------------------+---------------------------------------------------------------+
    |     commands     |                             format                            |
    +------------------+---------------------------------------------------------------+
    |       zadd       | zadd key member1 score1 [member2 score2 ...]                  |
    +------------------+---------------------------------------------------------------+
    |       zcard      | zcard key                                                     |
    +------------------+---------------------------------------------------------------+
    |      zrange      | zrange key start stop [WITHSCORES]                            |
    +------------------+---------------------------------------------------------------+
    |     zrevrange    | zrevrange key start stop [WITHSCORES]                         |
    +------------------+---------------------------------------------------------------+
    |   zrangebyscore  | zrangebyscore key min max [WITHSCORES][LIMIT offset count]    |
    +------------------+---------------------------------------------------------------+
    | zrevrangebyscore | zrevrangebyscore key max min [WITHSCORES][LIMIT offset count] |
    +------------------+---------------------------------------------------------------+
    | zremrangebyscore | zremrangebyscore key min max                                  |
    +------------------+---------------------------------------------------------------+
    |    zrangebylex   | zrangebylex key min max [LIMIT offset count]                  |
    +------------------+---------------------------------------------------------------+
    |  zrevrangebylex  | zrevrangebylex key max min [LIMIT offset count]               |
    +------------------+---------------------------------------------------------------+
    |  zremrangebylex  | zremrangebylex key min max                                    |
    +------------------+---------------------------------------------------------------+
    |      zcount      | zcount key                                                    |
    +------------------+---------------------------------------------------------------+
    |     zlexcount    | zlexcount key                                                 |
    +------------------+---------------------------------------------------------------+
    |      zscore      | zscore key member                                             |
    +------------------+---------------------------------------------------------------+
    |       zrem       | zrem key member1 [member2 ...]                                |
    +------------------+---------------------------------------------------------------+
    |      zclear      | zclear key                                                    |
    +------------------+---------------------------------------------------------------+
    |      zincrby     | zincrby key increment member                                  |
    +------------------+---------------------------------------------------------------+
    |     zpexpire     | zpexpire key int                                              |
    +------------------+---------------------------------------------------------------+
    |    zpexpireat    | zpexpireat key ts                                             |
    +------------------+---------------------------------------------------------------+
    |      zexpire     | zexpire key int                                               |
    +------------------+---------------------------------------------------------------+
    |     zexpireat    | zexpireat key ts                                              |
    +------------------+---------------------------------------------------------------+
    |       zpttl      | zpttl key                                                     |
    +------------------+---------------------------------------------------------------+
    |       zttl       | zttl key                                                      |
    +------------------+---------------------------------------------------------------+

### 4.6 Transaction

    +---------+---------+
    | command | support |
    +---------+---------+
    |  multi  | Yes     |
    +---------+---------+
    |   exec  | Yes     |
    +---------+---------+

### 4.7 Benchmark

基础benchmark数据，可以查看项目[wiki页面](https://github.com/yongman/tidis/wiki/Tidis-base-benchmark)

## 5. 参考
1. [GitHub - ideawu/ssdb: SSDB - A fast NoSQL database, an alternative to Redis](https://github.com/ideawu/ssdb)
2. [GitHub - siddontang/ledisdb: a high performance NoSQL powered by Go](https://github.com/siddontang/ledisdb)
3. [GitHub - ksarch-saas/ssdb: SSDB - A fast NoSQL database, an alternative to Redis](https://github.com/ksarch-saas/ssdb)
4. [GitHub - yongman/tidis: Distributed NoSQL database, Redis protocol compatible using tikv as backend](https://github.com/yongman/tidis)
5. [GitHub - JingchengLi/swapdb: https://github.com/JingchengLi/swapdb/wiki](https://github.com/JingchengLi/swapdb)
6. [GitHub - pingcap/tidb: TiDB is a distributed HTAP database compatible with the MySQL protocol](https://github.com/pingcap/tidb)
7. [GitHub - pingcap/tikv: Distributed transactional key value database powered by Rust and Raft](https://github.com/pingcap/tikv)
8. [tikv - GoDoc](https://godoc.org/github.com/pingcap/tidb/store/tikv)
9. [GitHub - apple/foundationdb: FoundationDB - the open source, distributed, transactional key-value store](https://github.com/apple/foundationdb)
10. [FoundationDB | Home](https://www.foundationdb.org/)
11. [Benchmarking — FoundationDB 5.1](https://apple.github.io/foundationdb/benchmarking.html)
