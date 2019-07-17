---
title: Redis冷热数据分离混合存储实现 - 读写任务异步化
categories: [存储]
date: 2018-09-30 17:59:14
---

redis的单线程处理模型，对于客户端请求都会变成串行处理，所以也不存在数据竞争的问题。对于冷数据需要的经过磁盘IO，这对于redis的高并发模型影响非常大。如果采用同步处理方式，磁盘IO由主线程处理，在进行磁盘IO的时候，后续的所有客户端请求会全部pending，无法进行处理，带来的问题就是单个磁盘IO影响整个服务的吞吐。

redis本身也有bio后台线程任务，主要是处理aof落盘、lazyfree逻辑和关闭大文件fd。要实现bio线程任务不会和主线程产生竞争，才能避免由于低频率的后台任务导致高频率的主线程数据访问竞争引入的开销。

对于aof，主线程写入内存buffer，按策略将buffer数据写入文件fd，提交异步任务对fd刷盘。

对于大的内存对象，redis也采用了lazyfree机制，在删除对象的时候，首先将对象从dict中unlink，客户端就无法访问这块内存了，unlink的操作是在主线程中完成的，将unlink的内存对象交给bio线程执行释放操作，不会存在资源竞争。

冷数据从磁盘load和dump操作也采用bio的操作方式。为了记录客户端和在执行bio操作的key的对应关系，需要引入几个变量。

- db->swapping_keys
 
dict类型，key是正在load或dump的的key，value是一个list，记录想要访问这个key的客户端连接，用于key就绪后处理响应客户端。

- client->bpop.swapping_keys

dict类型，仅仅用来记录该client阻塞的key， value是NULL，当有一个客户端同时阻塞在多个key上，dict记录的所有key全部就绪后才会将客户端就绪。

- db->ready_swap_keys

dict类型，用于记录db->swapping_keys中的在bio线程中处理完成就绪的key，处理完成后将key加入dict，加快就绪key查询速度。

- server.ready_swap_key

list类型，bio线程处理完成的任务在增加db->ready_swap_keys同时，会将db和key封装成readyList，加入list列表，后续在beforeSleep中对就绪key进行处理。

为了避免竞争，要能够保证所有对dict的更新操作需要在主线程完成，bio线程只做耗时的读写磁盘。

一个key在swap过程中，所有的客户端访问到这个key都会block，所以相当于对这个key进行了加锁，保证key不会被访问，load的过程中，主线程会将这个key和空的对象增加到主dict，将dict中的value的地址交给bio线程，bio线程更value的内容。dump的过程类似，主线程将key和value对象引用加一，防止内存释放，然后交给bio线程对内存数据落盘，然后将引用减一。

访问冷数据过程：

1. 查看key是冷热数据，热数据直接返回内存数据；
2. 如果key正在swap操作，直接block客户端；
3. key没有在swap的后台任务列表，并且是冷数据，从call命理返回，将key加入bio任务BLOCKED_LOAD队列；
4. 后台任务处理完成，加入ready_keys，交给主线程处理；
5. 主线程判断就绪key所阻塞的客户端，并且对于的客户端没有其他block的key，需要将客户端unblock，重新进入call，进行命令处理。

上面过程中，bio线程和主线程对于ready_keys等几个变量的操作需要防止竞争访问需要加锁，但这部分开销不大，尽量保证bio线程中的临界区小，不会对主线程产生影响。


