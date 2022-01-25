---
title: RocksDB参数调优
categories: ["存储"]
tags: ["RocksDB"]
date: 2018-12-05 17:58:49
---

RocksDB对比LevelDB暴露了很多参数来适应更多的应用场景，带来的好处就是可以通过tuning使系统性能达到最大，当然，如果tuning不合理会有相反的后果。在Facebook内部，RocksDB既能用在普通机械盘，生产环境也有跑在SSD和全内存的场景下。

当决定在生产环境使用RocksDB的话需要对RocksDB的原理和对应参数的含义非常清楚，修改参数之前要知道修改这个参数所带来的后果。

官方github [wiki](https://github.com/facebook/rocksdb/wiki/RocksDB-Tuning-Guide)有专门的RocksDB调优的页面，可以对调优方面有更深入的理解。

1. Amplification factors

其实对数据库系统的调优很大一部分工作是在写放大、读放大和空间放大这三个放大因子之间做取舍。

- 写放大

平均写入1个字节，引擎中在数据的生命周期内实际会写入n个字节，其写放大率是n。也就是如果在业务方写入速度是10MB/s，在引擎端或者操作系统层面能观察到的数据写入速度是30MB/s，这时，系统的写放大率就是3。写放大过大会制约系统的实际吞吐。并且对于SSD来说，越大的写放大，也会导致SSD寿命缩短。

- 读放大

一个小的读请求，系统所需要读去n个页面来完成查询，其读放大率是n。逻辑上的读操作可能会命中引擎内部的cache或者文件系统cache，命中不了cache就会进行实际的磁盘IO，命中cache的读取操作的代价虽然很低，但是也会消耗cpu。

- 空间放大

就是平均存储1个字节的数据，在存储引擎内部所占用的磁盘空间n和字节，其空间放大是n。比如刚刚写入10MB的数据，大小在磁盘上实际占用了100MB，这是空间放大率就是10。空间放大和写放大在调优的时候往往是排斥的，空间放大越大，那么数据可能不需要频繁的compaction，其写放大就会降低；如果空间放大率设置的小，那么数据就需要频繁的compaction来释放存储空间，导致写放大增大。

所以调优就是根据实际应用场景进行多方面权衡的一个过程。

2. Statistics

statistics是RocksDB用来统计系统性能和吞吐信息的功能，开启它可以更直接的提供性能观测数据，能快速发现系统的瓶颈或系统运行状态，由于统计信息在引擎内的各类操作都会设置很多的埋点，用来更新统计信息，但是开启statistics会增加5%到10%的额外开销。

```
    ** Compaction Stats **
    Level Files  Size(MB) Score Read(GB)  Rn(GB) Rnp1(GB) Write(GB) Wnew(GB) Moved(GB) W-Amp Rd(MB/s) Wr(MB/s) Comp(sec) Comp(cnt) Avg(sec) Stall(sec) Stall(cnt) Avg(ms)     KeyIn   KeyDrop
    -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
    L0      2/0        15   0.5     0.0     0.0      0.0      32.8     32.8       0.0   0.0      0.0     23.0    1457      4346    0.335       0.00          0    0.00             0        0
    L1     22/0       125   1.0   163.7    32.8    130.9     165.5     34.6       0.0   5.1     25.6     25.9    6549      1086    6.031       0.00          0    0.00    1287667342        0
    L2    227/0      1276   1.0   262.7    34.4    228.4     262.7     34.3       0.1   7.6     26.0     26.0   10344      4137    2.500       0.00          0    0.00    1023585700        0
    L3   1634/0     12794   1.0   259.7    31.7    228.1     254.1     26.1       1.5   8.0     20.8     20.4   12787      3758    3.403       0.00          0    0.00    1128138363        0
    L4   1819/0     15132   0.1     3.9     2.0      2.0       3.6      1.6      13.1   1.8     20.1     18.4     201       206    0.974       0.00          0    0.00      91486994        0
    Sum  3704/0     29342   0.0   690.1   100.8    589.3     718.7    129.4      14.8  21.9     22.5     23.5   31338     13533    2.316       0.00          0    0.00    3530878399        0
    Int     0/0         0   0.0     2.1     0.3      1.8       2.2      0.4       0.0  24.3     24.0     24.9      91        42    2.164       0.00          0    0.00      11718977        0
    Flush(GB): accumulative 32.786, interval 0.091
    Stalls(secs): 0.000 level0_slowdown, 0.000 level0_numfiles, 0.000 memtable_compaction, 0.000 leveln_slowdown_soft, 0.000 leveln_slowdown_hard
    Stalls(count): 0 level0_slowdown, 0 level0_numfiles, 0 memtable_compaction, 0 leveln_slowdown_soft, 0 leveln_slowdown_hard
    
    ** DB Stats **
    Uptime(secs): 128748.3 total, 300.1 interval
    Cumulative writes: 1288457363 writes, 14173030838 keys, 357293118 batches, 3.6 writes per batch, 3055.92 GB user ingest, stall micros: 7067721262
    Cumulative WAL: 1251702527 writes, 357293117 syncs, 3.50 writes per sync, 3055.92 GB written
    Interval writes: 3621943 writes, 39841373 keys, 1013611 batches, 3.6 writes per batch, 8797.4 MB user ingest, stall micros: 112418835
    Interval WAL: 3511027 writes, 1013611 syncs, 3.46 writes per sync, 8.59 MB written
```

3. Parallelism options

LSM架构的引擎，后台主要有两种线程，flush和compaction。在RocksDB中，两种线程有不同的优先级，flush优先级高，compaction优先级低。为了充分利用多核，RocksDB中的两种线程可以配置线程数。

**max_background_flushes** 是后台memtable dump成sstable的并发线程数。默认是1，但是当使用多个`column family`时，内部会存在多个memtable，可能会同时发生flush，如果线程是1，在写入量大的情况下，可能会导致flush不及时，出现无法写入的情况。

**max_background_compactions** 是后台sstable flush线程，因为一般RocksDB会有多层，涉及多个sstable文件，并发compaction会加快compaction的速度，如果compaction过慢，达到`soft_pending_compaction_bytes_limit`会发生阻塞，达到`hard_pending_compaction_bytes`会停写。

4. General options

**filter_policy** 这个就是每个sstable的bloom filter，使用bloom filter可以大幅减少不必要的磁盘IO。在bits_per_key为10的情况下，bloom filter错误率估计为1%，也就是存在如下情况：有1%的概率出现错误，key实际上不存在，但是在bloom filter查询的结果是存在的。这种情况导致会有1%的不必要的磁盘IO。

当然bits_per_key越大，bloom filter误判率越低，但是占用的内存和写放大会相应增加。

**block_cache** 可以配置大小的LRU cache，用来缓存未经过压缩的block。由于访问cache需要加锁访问，当大部分数据都在cache中时，多线程并发访问cache可能会出现锁竞争的瓶颈，所以LRU cache还有一个shard_bits参数，将LRU cache分片，其实就是对锁进行拆分，不同分片的cache不会出现竞争。默认shard_bits是6，那么cache的shard数目就是`2^6=64`。

**allow_os_buffer** 操作系统buffer是用来缓存sstable文件的cache，sstable在文件中是压缩的，所以操作系统buffer是对磁盘上的sstable文件block的cache。

**max_open_files** 最大打开文件数。RocksDB对于打开文件的句柄也会放在cache中，当sstable文件过多，超过max_open_files限制后，会将cache中淘汰的文件句柄强制关闭，下次读取文件的时候再次打开。

**table_cache_numshardbits** 和block_cache中的shard_bits作用一致，主要是为了拆分互斥锁。

**block_size** RocksDB中sstable文件的由block组成，block也是文件读写的最小逻辑单位，当读取一个很小的key，其实会读取一个block到内存，然后查找数据。默认的block_size大小为4KB。每个sstable文件都会包含索引的block，用来加快查找。所以block_size越大，index就会越少，也会相应的节省内存和存储空间，降低空间放大率，但是会加剧读放大，因为读取一个key实际需要读取的文件大小随之增加了。

5. Sharing cache and thread pool

在一个进程中启动多个RocksDB实例，可以共享cache和线程池，暂时没有这方面需求。

6. Flushing options

RocksDB中的写入在写WAL后会先写入memtable，memtable达到特定大小后会转换成immutable membtale，immutable会flush成sstable。

**write_buffer_size** 配置单个memtable的大小，当memtable达到指定大小，会自动转换成immutable memtable并且新创建一个memtable。

**max_write_buffer_number** 指定一个RocksDB中memtable和immutable memtable总的数量。当写入速度过快，或者flush线程速度较慢，出现memtable数量超过了指定大小，请求会无法写入。

**min_write_buffer_number_to_merge** immutable memtable在flush之前先进行合并，比如参数设置为2，当一个memtable转换成immutable memtable后，RocksDB不会进行flush操作，等到至少有2个后才进行flush操作。这个参数调大能够减少磁盘写的次数，因为多个memtable中可能有重复的key，在flush之前先merge后就避免了旧数据刷盘；但是带来的问题是每次数据查找，当memtable中没有对应数据，RocksDB可能需要遍历所有的immutable memtable，会影响读取性能。

下面是官方wiki配置的一个例子：


>write_buffer_size = 512MB;

>max_write_buffer_number = 5;

>min_write_buffer_number_to_merge = 2;


按上面的配置，如果数据写入速度是16MB/s，每32秒会生成一个新的memtable，每64秒会生成两个memtable进行merge和flush操作。如果flush速度很快，memtable占用的内存应该小于1.5G，当磁盘IO繁忙，flush速度慢，最多会有5个memtable，占用内存达到2.5G，后续就无法写入。

7. Level Style Compaction

具体的compaction逻辑查看代码实现或者自行搜索。下面主要看一下涉及到compaction的配置项。

**level0_file_num_compaction_trigger** L0达到指定个数的sstable后，触发compaction L0->L1。所以L0稳定状态下大小为`write_buffer_size*min_write_buffer_number_to_merge*level0_file_num_compaction_trigger`。

**max_bytes_for_level_base** L1的总大小，L1的大小建议设置成和L0大小一致，提升L0->L1的compaction效率。

**max_bytes_for_level_multiplier** L(n+1)总大小=L(n) * max_bytes_for_level_multiplier，默认值为10，下一层是上一层容量的10倍。

**target_file_size_base** L1层单个sstable文件的大小。建议该参数设置为`max_bytes_for_level_base/10`，能够保证L1层的sstable文件数量为10个。

**target_file_size_multiplier** 下一层文件大小是上一层文件大小的倍数，默认值为1，也就是不同层的单个sstable文件大小一致，调大这个值可以减少每层的文件数量。

**compression_per_level** 配置各Level的压缩模式，由于L0的特殊性，L0的merge操作可能涉及L1的全部sstable，为了提高性能，L0和L1不要启用压缩，在其他更高层上启用压缩。

**num_levels** RocksDB中的层次数量，默认是7。如果L0大小有512MB，6层能容纳`512M+512M+5G+50G+500G+5T`，如果配置是7，在数据量少于前面计算的5T+的数据之前，最后一层是不会被使用的。如果num_levels配置为6，那么最下面一层数据量会大于5T。

8. Write stalls

RocksDB在flush或compaction速度来不及处理新的写入，会启动自我保护机制，延迟写或者禁写。主要有几种情况：

- Too many memtables

**写限速**：如果max_write_buffer_number大于3，将要flush的memtables大于等于max_write_buffer_number-1，write会被限速。

**禁写**：memtable个数大于等于max_write_buffer_number，触发禁写，等到flush完成后允许写入。


>Stopping writes because we have 5 immutable memtables (waiting for flush), max_write_buffer_number is set to 5  
>Stalling writes because we have 4 immutable memtables (waiting for flush), max_write_buffer_number is set to 5


- Too many level-0 SST files

**写限速**：L0文件数量达到level0_slowdown_writes_trigger，触发写限速。

**禁写**：L0文件数量达到level0_stop_writes_trigger，禁写。

> Stalling writes because we have 4 level-0 files

> Stopping writes because we have 20 level-0 files

- Too many pending compaction bytes

**写限速**：等待compaction的数据量达到soft_pending_compaction_bytes，触发写限速。

**禁写**：等待compaction的数据量达到hard_pending_compaction_bytes，触发禁写。

> Stalling writes because of estimated pending compaction bytes 500000000

> Stopping writes because of estimated pending compaction bytes 1000000000

当出现write stall时，可以按具体的系统的状态调整如下参数：

- 调大max_background_flushes
- 调大max_write_buffer_number
- 调大max_background_compactions
- 调大write_buffer_size
- 调大min_write_buffer_number_to_merge

9. Bloom filters and index

RocksDB中使用bloom filter来尽可能避免不必要的sstable访问，在RocksDB中bloom filter有两种：block-based filter和full filter。

- block-based filter

这种filter是保存在每个block内部，一个sstable file有多个block。所以每次访问sstable需要先访问一次block索引，找到对应的block后加载bloom filter或者是在cache中获取对应的filter查询。

- full filter

每个sstable文件只有一个filter，在访问sstable之前可以事先判断key的存在性，能够避免不存在key的索引访问。

一个典型的256MB的SST文件的index/filter的大小在0.5/5MB，比系统内默认的block大很多倍，这种场景对于block cache不友好，LRU cache是按block来进行置换，一部分block失效会导致重复读取多次SST加载index和filter。

对于上面大SST文件的index或者filter block，RocksDB支持两级index，切分index/filter。每次读取和cache的粒度变小，cache更高效。

**bloom_bits_per_key** 平均每个key需要的bloom filter空间。配置bloom filter的容量，默认是10，存在1%的判错概率。

**cache_index_and_filter_blocks_with_high_priority** 可以配置RocksDB高优先cache index和filter，优先剔除数据block。

**pin_l0_filter_and_index_blocks_in_cache** 使level 0中的SST的index和filter常驻cache。

10. 配置实例

- 存储介质flash

```
thread_pool 4
options.options.compaction_style = kCompactionStyleLevel;
options.write_buffer_size = 67108864; // 64MB
options.max_write_buffer_number = 3;
options.target_file_size_base = 67108864; // 64MB
options.max_background_compactions = 4;
options.level0_file_num_compaction_trigger = 8;
options.level0_slowdown_writes_trigger = 17;
options.level0_stop_writes_trigger = 24;
options.num_levels = 4;
options.max_bytes_for_level_base = 536870912; // 512MB
options.max_bytes_for_level_multiplier = 8;
```

- 全内存

```
options.allow_mmap_reads = true;
BlockBasedTableOptions table_options;
table_options.filter_policy.reset(NewBloomFilterPolicy(10, true));
table_options.no_block_cache = true;
table_options.block_restart_interval = 4;
options.table_factory.reset(NewBlockBasedTableFactory(table_options));
options.level0_file_num_compaction_trigger = 1;
options.max_background_flushes = 8;
options.max_background_compactions = 8;
options.max_subcompactions = 4;
options.max_open_files = -1;
ReadOptions.verify_checksums = false
```

