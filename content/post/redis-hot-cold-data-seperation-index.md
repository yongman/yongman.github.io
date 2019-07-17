---
title: Redis冷热数据分离混合存储实现 - index存储
categories: [存储]
date: 2018-09-20 18:04:45
---

为了实现冷热数据分离，热数据在内存，冷数据会置换到持久化存储，但是为了保证内存检索高效，会将所有key和频率统计信息保存在内存。冷数据的选择采用的选择算法和redis本身的数据淘汰选择算法一致，使用采样最优的方式。

为了将冷数据和热数据区分，方便冷数据采样，所以在每个db里增加了一个名为index的dict，用于保存key的index和频率统计信息。
![](https://raw.githubusercontent.com/yongman/i/img/picgo/DhBmufM.png)

### 换出策略

什么时候会触发冷数据的持久化来释放内存？需要根据应用场景定义不同的置换策略。

置换策略可以采用多种可配置方式：

- maxhotmemory

增加一项配置项，在内存数据达到阈值，触发数据强制置换，策略类似maxmemory删除策略，为了不影响原有数据淘汰逻辑，maxhotmemory要小于maxmemory，当达阈值后从db->dict采样key，根据访问的频率，选择最适合置换的key，然后持久化，逻辑如下：

1. 将对于的redis object decode成rawstring，调用RocksDB PUT接口，将数据存储到磁盘；
2. 将key和初始的频率访问信息添加到db->index;
3. 将key从db->dict清理掉，释放内存。

- 冷热数据百分比

根据冷热数据比例，在内存使用率到一定比例的前提下，由定时任务周期性的选择合适的key进行持久化，定时任务只是生成置换任务，交给IO线程异步处理，IO操作不阻塞主线程。持久化逻辑同上。


### 冷数据加载

冷数据加载采用lazy的处理方式，在读写key之前，会先去db->index中进行查询，如果在index中，则阻塞客户端，将key加入阻塞任务中，异步IO线程进行数据加载。加载逻辑：

1. 读取RockDB，生成redis object对象；
2. 从db->index中删除key；
3. 将新生成的value对象加入db->dict。

其中存在的问题就是当采用maxhotmemory策略，当usedmemory大于maxhotmemory，在进行写入操作会产生频繁的swap操作，所以还需要增加定时任务策略，尽量保证使usedmemory小于maxhotmemory；
当内存中全部存放的是key的index时，持续写入会使usedmemory达到maxmemory，触发数据淘汰逻辑，当配置了allkeys淘汰逻辑，server会按概率从dict或index中获取要淘汰的key。


### RocksDB API

```
/*
 * decode robj from key and value to rawstring
 * call rocksdb_put to save key-value pair
 */
int rocksAdd(rocksdb_t *rdb, rocksdb_writeoptions_t *options, robj *key, robj *val) {
    robj *dec;
    char *err = NULL;
    sds key_ptr, val_ptr;
    dec = getDecodedObject(val);


    key_ptr = key->ptr;
    val_ptr = dec->ptr;
    rocksdb_put(rdb, options, key_ptr, sdslen(key_ptr), val_ptr, sdslen(val_ptr), &err);
    decrRefCount(dec);

    if (err != NULL) {
        serverLog(LL_WARNING, "rocksAdd failed, %s", err);
        return C_ERR;
    }

    return C_OK;
}

/*
 * decode robj from key to rawstring
 * call rocksdb_get to get value from rocksdb of the key
 * encode the value to robj
 * */
robj *rocksGet(rocksdb_t *rdb, rocksdb_readoptions_t *options, robj *key) {
    robj *val;
    char *err = NULL;
    size_t vallen;
    sds key_ptr;
    char *val_raw;

    key_ptr = key->ptr;
    val_raw = rocksdb_get(rdb, options, key_ptr, sdslen(key_ptr), &vallen, &err);
    if (err != NULL) {
        serverLog(LL_WARNING, "rocksGet failed, %s", err);
        return NULL;
    }
    if (val_raw == NULL) {
        // not found
        return NULL;
    } else {
        val = createRawStringObject(val_raw, vallen);
        // try encoding obj
        val = tryObjectEncoding(val);

        free(val_raw);
    }

    return val;
}

int rocksDel(rocksdb_t *rdb, rocksdb_writeoptions_t *options, robj *key) {
    char *err = NULL;
    sds key_ptr;

    key_ptr = key->ptr;

    rocksdb_delete(rdb, options, key_ptr, sdslen(key_ptr), &err);
    if (err != NULL) {
        serverLog(LL_WARNING, "rocksDel failed, %s", err);
        return C_ERR;
    }

    return C_OK;

}

void rocksOpen() {
    if (server.rdb) return;


    server.r_options = rocksdb_options_create();
    server.r_read_options = rocksdb_readoptions_create();
    server.r_write_options = rocksdb_writeoptions_create();
    // default config
    .....
    
    /* rocksdb block based options */
    rocksdb_block_based_table_options_t *b_options = rocksdb_block_based_options_create();
    rocksdb_block_based_options_set_cache_index_and_filter_blocks(b_options, 0);

    rocksdb_options_set_block_based_table_factory(server.r_options, b_options);

    char *err = NULL;
    server.rdb = rocksdb_open(server.r_options, "/tmp/rocksdb_redis", &err);
    if (err != NULL) {
        serverLog(LL_WARNING, "open rocksdb failed, %s", err);
    }
}

void rocksClose() {
    if (!server.r_options) {
        rocksdb_options_destroy(server.r_options);
    }
    if (!server.r_read_options) {
        rocksdb_readoptions_destroy(server.r_read_options);
    }
    if (!server.r_write_options) {
        rocksdb_writeoptions_destroy(server.r_write_options);
    }
    if (!server.rdb) {
        rocksdb_close(server.rdb);
    }
}
```

