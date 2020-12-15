---
title: Tidis - Distributed transactional NoSQL database, Redis protocol compatible using tikv as backend
date: 2018-08-10 13:58:30
---

Github repo: [https://github.com/yongman/tidis](https://github.com/yongman/tidis)
<a class="github-button" href="https://github.com/yongman/tidis" data-icon="octicon-star" data-size="large" data-show-count="true" aria-label="Star yongman/tidis on GitHub">Star</a>
<script async defer src="https://buttons.github.io/buttons.js"></script>

# What is Tidis?

[Tidis](https://github.com/yongman/tidis) is a Distributed NoSQL database, providing a Redis protocol API (string, list, hash, set, sorted set), written in Go.

Tidis is like [TiDB](https://github.com/pingcap/tidb) layer, providing protocol transform and data structure compute, powered by [TiKV](https://github.com/pingcap/tikv) backend distributed storage which use Raft for data replication and 2PC for distributed transaction.

## Features

* Redis protocol compatible
* Linear scale-out ability
* Storage and computation separation
* Data safety, no data loss, Raft replication
* Transaction support

Any pull requests are welcomed.

## Architecture

*Architechture of tidis*

![architecture](https://raw.githubusercontent.com/yongman/tidis/master/docs/tidis-arch.png)

*Architechture of tikv*

![](https://pingcap.com/images/blog/TiKV_%20Architecture.png)
- Placement Driver (PD): PD is the brain of the TiKV system which manages the metadata about Nodes, Stores, Regions mapping, and makes decisions for data placement and load balancing. PD periodically checks replication constraints to balance load and data automatically.
- Node: A physical node in the cluster. Within each node, there are one or more Stores. Within each Store, there are many Regions.
- Store: There is a RocksDB within each Store and it stores data in local disks.
- Region: Region is the basic unit of Key-Value data movement and corresponds to a data range in a Store. Each Region is replicated to multiple Nodes. These multiple replicas form a Raft group. A replica of a Region is called a Peer.

## 1. Run tidis server with docker-compose in one command

```
git clone https://github.com/yongman/tidis-docker-compose.git
cd tidis-docker-compose/
docker-compose up -d
```

Or follow [tidis-docker-compose](https://github.com/yongman/tidis-docker-compose) guide to run with `docker-compose`

## 2. Build or Docker mannualy

### Build from source

```
git clone https://github.com/yongman/tidis.git
cd tidis && make
```

### Pull from docker

```
docker pull yongman/tidis
```

### Run TiKV cluster for test

Use `docker run tikv` for test, just follow [PingCAP official guide](https://github.com/pingcap/docs/blob/master/op-guide/docker-deployment.md), you just need to deploy PD and TiKV servers, Tidis will take the role of TiDB.

### Run Tidis or docker

#### Run tidis from executable file

```
bin/tidis-server -conf config.toml
```

### Run tidis from docker

```
docker run  -d --name tidis -p 5379:5379 -v {your_config_dir}:/data yongman/tidis -conf="/data/config.toml"
```

## 3. Client request

```
redis-cli -p 5379
127.0.0.1:5379> get a
"1"
127.0.0.1:5379> lrange l 0 -1
1) "6"
2) "5"
3) "4"
127.0.0.1:5379> zadd zzz 1 1 2 2 3 3 4 4
(integer) 4
127.0.0.1:5379> zcard zzz
(integer) 4
127.0.0.1:5379> zincrby zzz 10 1
(integer) 11
127.0.0.1:5379> zrange zzz 0 -1 withscores
1) "2"
2) "2"
3) "3"
4) "3"
5) "4"
6) "4"
7) "1"
8) "11"
```


## Already supported commands

### Keys

    +-----------+-------------------------------------+
    |  pexpire  | pexpire key int                     |
    +-----------+-------------------------------------+
    | pexpireat | pexpireat key timestamp(ms)         |
    +-----------+-------------------------------------+
    |   expire  | expire key int                      |
    +-----------+-------------------------------------+
    |  expireat | expireat key timestamp(s)           |
    +-----------+-------------------------------------+
    |    pttl   | pttl key                            |
    +-----------+-------------------------------------+
    |    ttl    | ttl key                             |
    +-----------+-------------------------------------+
    |    type   | type key                            |
    +-----------+-------------------------------------+

### String

    +-----------+-------------------------------------+
    |  command  |               format                |
    +-----------+-------------------------------------+
    |    get    | get key                             |
    +-----------+-------------------------------------+
    |    set    | set key value [EX sec|PX ms][NX|XX] | 
    +-----------+-------------------------------------+
    |   getbit  | getbit key offset                   |
    +-----------+-------------------------------------+
    |   setbit  | setbit key offset value             |
    +-----------+-------------------------------------+
    |    del    | del key1 key2 ...                   |
    +-----------+-------------------------------------+
    |    mget   | mget key1 key2 ...                  |
    +-----------+-------------------------------------+
    |    mset   | mset key1 value1 key2 value2 ...    |
    +-----------+-------------------------------------+
    |    incr   | incr key                            |
    +-----------+-------------------------------------+
    |   incrby  | incr key step                       |
    +-----------+-------------------------------------+
    |    decr   | decr key                            |
    +-----------+-------------------------------------+
    |   decrby  | decrby key step                     |
    +-----------+-------------------------------------+
    |   strlen  | strlen key                          |
    +-----------+-------------------------------------+


### Hash

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

### List

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

### Set

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

### Sorted set

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
    |      zincrby     | zincrby key increment member                                  |
    +------------------+---------------------------------------------------------------+

### Transaction

    +---------+---------+
    | command | support |
    +---------+---------+
    |  multi  | Yes     |
    +---------+---------+
    |   exec  | Yes     |
    +---------+---------+

### Server & Connections

    +-----------+---------------+
    | command  	| format     	|
    +-----------+---------------+
    | flushdb  	| flushdb id 	|
    +-----------+---------------+
    | flushall 	| flush      	|
    +-----------+---------------+
    | select   	| select id  	|
    +-----------+---------------+

## Benchmark

[base benchmark](https://github.com/yongman/tidis/wiki/Tidis-base-benchmark)
### Environment

- OS: Debian 8.6
- Kernel: 3.16
- Memory: 250GB
- Processor: Intel(R) Xeon(R) CPU E5-2670 v3 @ 2.30GHz * 48
- Disk: SATA (SSD is recommended)

## Configuration

- One pd server (run with docker)
- One tikv server (run with docker), configuration as follows tikv.conf

  ```
  log-level = "info"
  [server]
  addr = "ip:20160"
  [storage]
  data-dir = "tikv"
  [pd]
  endpoints = ["ip:2379"]
  [metric]
  interval = "1500s"
  address = ""
  job = "tikv"
  [raftstore]
  sync-log = false
  region-max-size = "384MB"
  region-split-size = "256MB"
  region-split-check-diff = "32MB"
  [rocksdb]
  max-manifest-file-size = "20MB"
  [rocksdb.defaultcf]
  block-size = "64KB"
  compression-per-level = ["no", "no", "lz4", "lz4", "lz4", "zstd", "zstd"]
  write-buffer-size = "128MB"
  max-write-buffer-number = 5
  level0-slowdown-writes-trigger = 20
  level0-stop-writes-trigger = 36
  max-bytes-for-level-base = "512MB"
  target-file-size-base = "32MB"
  [rocksdb.writecf]
  compression-per-level = ["no", "no", "lz4", "lz4", "lz4", "zstd", "zstd"]
  write-buffer-size = "128MB"
  max-write-buffer-number = 5
  min-write-buffer-number-to-merge = 1
  max-bytes-for-level-base = "512MB"
  target-file-size-base = "32MB"
  [raftdb]
  compaction-readahead-size = "2MB"
  [raftdb.defaultcf]
  compression-per-level = ["no", "no", "lz4", "lz4", "lz4", "zstd", "zstd"]
  write-buffer-size = "128MB"
  max-write-buffer-number = 5
  min-write-buffer-number-to-merge = 1
  max-bytes-for-level-base = "512MB"
  target-file-size-base = "32MB"
  block-cache-size = "256MB"
  ```
  
- One tidis server with default configuration

  ```
  bin/tidis-server -backend ip:2379
  ```
  
### Use redis-benchmark with transaction support

1.  Let's start with one client concurrency

* GET

`redis-benchmark -p 7379 -t GET -r 100000000 -n 10000 -c 1`

```
====== GET ======
  10000 requests completed in 5.64 seconds
  1 parallel clients
  3 bytes payload
  keep alive: 1

99.76% <= 1 milliseconds
100.00% <= 2 milliseconds
1773.05 requests per second
```
* SET

`redis-benchmark -p 7379 -t SET -r 100000000 -n 1000 -c 1`

```
====== SET ======
  1000 requests completed in 2.23 seconds
  1 parallel clients
  3 bytes payload
  keep alive: 1

0.10% <= 1 milliseconds
12.80% <= 2 milliseconds
99.30% <= 3 milliseconds
99.90% <= 4 milliseconds
100.00% <= 8 milliseconds
448.83 requests per second
```

If you do this, you will find tidis is _quite slow_.

Why? The biggest reason is every write would create a transaction, and the transaction timestamp must be obtain from pd servers, and the request must be encoded/decoded and make a rpc request to tikv server and tikv server use raft for replication and twe-phase commit for distributed transaction, which cause the latancy higher. So throughput of one connection seems quit small.

2. Start with 50 concurrency

* GET

`redis-benchmark -p 7379 -t GET -r 100000000 -n 10000 -c 50`

```
====== GET ======
  10000 requests completed in 0.52 seconds
  50 parallel clients
  3 bytes payload
  keep alive: 1

28.86% <= 1 milliseconds
43.97% <= 2 milliseconds
67.82% <= 3 milliseconds
82.37% <= 4 milliseconds
88.53% <= 5 milliseconds
93.77% <= 6 milliseconds
96.76% <= 7 milliseconds
98.05% <= 8 milliseconds
98.98% <= 9 milliseconds
99.44% <= 10 milliseconds
99.71% <= 11 milliseconds
99.87% <= 12 milliseconds
99.93% <= 13 milliseconds
99.99% <= 14 milliseconds
100.00% <= 15 milliseconds
19379.85 requests per second
```

* SET

`redis-benchmark -p 7379 -t SET -r 100000000 -n 10000 -c 50`

```
====== SET ======
  10000 requests completed in 1.21 seconds
  50 parallel clients
  3 bytes payload
  keep alive: 1

0.01% <= 1 milliseconds
0.03% <= 2 milliseconds
0.70% <= 3 milliseconds
9.10% <= 4 milliseconds
31.07% <= 5 milliseconds
57.32% <= 6 milliseconds
77.68% <= 7 milliseconds
89.09% <= 8 milliseconds
94.78% <= 9 milliseconds
97.63% <= 10 milliseconds
98.49% <= 11 milliseconds
98.86% <= 12 milliseconds
99.05% <= 13 milliseconds
99.09% <= 14 milliseconds
99.12% <= 15 milliseconds
99.23% <= 16 milliseconds
99.29% <= 17 milliseconds
99.45% <= 18 milliseconds
99.60% <= 19 milliseconds
99.80% <= 20 milliseconds
99.93% <= 21 milliseconds
99.99% <= 22 milliseconds
100.00% <= 24 milliseconds
8271.30 requests per second
```

3. Let's write batch in one transaction

* One client with batch 10 writes in one transaction

`redis-benchmark -p 7379 -t SET -r 100000000 -n 10000 -c 1 -T 10`

```
====== SET ======
  10000 requests completed in 2.54 seconds
  1 parallel clients
  3 bytes payload
  keep alive: 1

3933.91 requests per second
```

* 50 concurrent clients with batch 10 writes in one transaction

`redis-benchmark -p 7379 -t SET -r 100000000 -n 100000 -c 50 -T 10`

```
====== SET ======
  100000 requests completed in 1.55 seconds
  50 parallel clients
  3 bytes payload
  keep alive: 1

64516.13 requests per second
```

* 1000 concurrent clients with batch 100 writes in one transaction

`redis-benchmark -p 7379 -t SET -r 100000000 -n 1000000 -c 1000 -T 100`

```
====== SET ======
  1000000 requests completed in 10.19 seconds
  1000 parallel clients
  3 bytes payload
  keep alive: 1

89.90% <= 1 milliseconds
89.94% <= 4 milliseconds
89.96% <= 5 milliseconds
89.99% <= 7 milliseconds
90.00% <= 21 milliseconds
90.01% <= 23 milliseconds
90.02% <= 24 milliseconds
90.03% <= 25 milliseconds
90.06% <= 26 milliseconds
90.15% <= 27 milliseconds
90.29% <= 28 milliseconds
90.52% <= 29 milliseconds
90.95% <= 30 milliseconds
91.53% <= 31 milliseconds
92.44% <= 32 milliseconds
94.50% <= 33 milliseconds
96.27% <= 34 milliseconds
97.32% <= 35 milliseconds
98.27% <= 36 milliseconds
98.87% <= 37 milliseconds
99.16% <= 38 milliseconds
99.31% <= 39 milliseconds
99.51% <= 40 milliseconds
99.58% <= 41 milliseconds
99.64% <= 42 milliseconds
99.73% <= 43 milliseconds
99.83% <= 44 milliseconds
99.90% <= 45 milliseconds
99.92% <= 46 milliseconds
99.94% <= 47 milliseconds
99.95% <= 48 milliseconds
99.98% <= 49 milliseconds
99.99% <= 50 milliseconds
100.00% <= 50 milliseconds
98183.60 requests per second
```

This is a base benchmark for now, more scenarios bench data need be added.

## License

Tidis is under the MIT license. See the [LICENSE](https://github.com/yongman/tidis/blob/master/LICENSE) file for details.
