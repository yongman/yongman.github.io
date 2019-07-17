---
title: Leto - 基于raft快速实现一个key-value存储系统
categories: [分布式]
date: 2018-07-30 16:16:17
---

[Leto基于raft和redis协议的key value存储系统](https://github.com/yongman/leto)

`raft`是一种类似于`paxos`的一致性算法，但是相对`paxos`，`raft`更易于简单，易于理解，所以工程实现也比较多。对于`golang`来说，经过工业验证的是hashicorp(consul)的`raft`库和etcd中实现的`raft`库。

## raft协议

1. [raft可视化](http://thesecretlivesofdata.com/raft/)
2. [Raft Consensus Algorithm](https://raft.github.io/)
3. [A key value storage example powered by hashicorp raft and BadgerDB](https://github.com/yongman/leto)

## raft库选择

大体看了下etcd raft和hashicorp raft的实现，感觉上来说hashicorp的`raft`包装更好，后端使用了封装好的`boltdb`作为LogStore和StableStore，而etcd的`raft`库可定制性更强，封装没有那么完整，读起来也比较费力。作为初次使用`raft`，选择hashicorp raft感觉会更容易上手。

## 快速实现一个分布式key value存储系统

既然是分布式，其具备的分布式系统的基本特性，该系统取名[勒托](https://zh.wikipedia.org/zh-hans/%E5%8B%92%E6%89%98),来自希腊神话，阿波罗之母。

`Github`地址：
[https://github.com/yongman/leto](https://github.com/yongman/leto)。

### 1. 特性

- 容错性
- 集群节点动态变化
- 数据安全性
- 可以支持多种后端引擎
- snapshot支持
- 集群线性扩展(待实现)

### 2. 目的

虽然从零造轮子不是什么好主意，为了更好的学习`raft`，在实践中摸索，对于初学者对于加深理解`raft`很有帮助。

### 3. 具体实现

定义一个`store`，其中包含了`raft`实例和`fsm`实例。

- `fsm`实例就是`raft`中保存状态的组件，它完成了状态机生成快照、恢复快照、和回放`raft log`来更新状态机。
- `raft`实例就是创建的hashicorp `raft`对象，创建时会将上面的`fsm`实例传入，用于自定义实现存储的过程。

生成快照采用了`google protobuf`协议序列化数据并打包，数据通过`raft.FSMSnapshot`接口的`Persist`实现数据打包并持久化磁盘。

**Store对象**

```
type Store struct {
	RaftDir  string
	RaftBind string

	raft *raft.Raft // The consensus mechanism
	fsm  *fsm

	logger *log.Logger
}
```

**Store提供的api**

```
// 生成store对象
func (s *Store) Open(bootstrap bool, localID string) error

// 提供的上层数据操作接口，调用底层raft或者fsm接口
func (s *Store) Get(key string) (string, error)
func (s *Store) Set(key, value string) error
func (s *Store) Delete(key string) error

// 集群成员变更处理接口
// 新成员加入集群
func (s *Store) Join(nodeID, addr string) error
// 成员退出集群
func (s *Store) Leave(nodeID string) error

// 生成快照，用户触发
func (s *Store) Snapshot() error
```

**fsm对象**

```
type fsm struct {
	db DB

	logger *log.Logger
}
```

**fsm提供的api**

```
// 状态查询
func (f *fsm) Get(key string) (string, error)
// 状态应用
func (f *fsm) Apply(l *raft.Log) interface{}

// 生成快照，这里只是返回了一个实现了FSMSnapshot接口的对象
func (f *fsm) Snapshot() (raft.FSMSnapshot, error)
func (f *fsm) Restore(rc io.ReadCloser) error
func (f *fsm) Close() error
```

**FSMSnapshot对象**

```
type fsmSnapshot struct {
	db     DB
	logger *log.Logger
}

// 具体实现fsm状态持久化逻辑，一般是通过生成db快照，然后遍历db，生成kv对然后序列化
func (f *fsmSnapshot) Persist(sink raft.SnapshotSink) error
```

**后端引擎接口**

```
type DB interface {
	Get(key []byte) ([]byte, error)
	Set(key, value []byte) error
	Delete(key []byte) (bool, error)

	SnapshotItems() <-chan DataItem

	Close()
}

type DataItem interface{}
```

**server api**

server api采用redis协议，用户可以使用任何redis客户端来与leto通信。

**组建集群**

第一个节点启动采用`bootstrap`方式

```
bin/leto -id id1 -raftdir ./id1
```

后续节点启动后指定第一个节点的`server api`地址，将本节点信息通过`join`命令发送给`bootstrap`节点，`bootstrap`节点会将刚启动的节点加入`raft`集群。

```
bin/leto -id id2 -raftdir ./id2 -listen ":6379" -raftbind ":16379" -join "127.0.0.1:5379"
```

启动第三个节点

```
bin/leto -id id3 -raftdir ./id3 -listen ":7379" -raftbind ":17379" -join "127.0.0.1:5379"
```

### 4. 目前支持的命令

- GET
- SET
- DELETE
- JOIN
- LEAVE
- PING
- SNAPSHOT
