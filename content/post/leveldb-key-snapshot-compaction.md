---
title: LevelDB中snapshot和compaction
categories: [存储]
date: 2018-08-03 19:23:54
---

本文简单记录了leveldb中有关key的种类、memtable的操作、snapshot的原理和Compaction。

### Key

- LookupKey: `klength|UserKey|tag`
- InternalKey: `UserKey|tag`，其中tag部分包括`SequenceNumber`和`ValueType`
- ParsedInternalKey: 包含`UserKey`、`SequenceNumber`和`ValueType`
- UserKey：用户存储的原始key

`userkey添加tag信息=>InternalKey 增加key长度=>LookupKey`

### Memtable

Memtable只提供了Get和Add，因为LevelDB的更新操作全部是追加写，包括delete操作。

Add操作的流程是先根据userKey和sequenceNumber生成InternalKey，然后插入skiplist。

Get操作流程是先根据传入的lookupKey从skiplist查找，查找到读取对应的value并解析tag，是`kTypeValue`就返回对应的value，是`kTypeDeletion`就返回NotFound。

### Snapshot

snapshot在LevelDB中的存在形式就是一个严格递增的SequenceNumber，LevelDB中最新的snapshot保存在
`versions_->LastSequence()`中。

userKey编码成Internalkey时，在userkey后面跟着sequenceNumber，同一个userKey的多个版本数据天然按新旧进行排序，当查看指定的snapshot版本的数据时，生成带有sequenceNumber的lookupKey，查找不大于指定snapshot版本的最大数据即可。

### Compaction

LevelDB中有两种类型compaction，Minor compaction和Major compaction。Compaction是后台线程来调度，读写线程中更新compaction统计信息，判断是否需要进行compaction，如果需要进行compaction就创建任务，加入任务队列。

- Minor compaction

后台任务检测到存在immutable memtable时，就会触发CompactMemTable，首先对currentVersion进行加引用，然后由versionSet生成一个新的sst file并dump数据到sst file，然后选择给sst file所放置的level。

一般情况下memtable dump的sst file都放在0 level，但是当第0层中的所有sst file都不包含memtable中的key的range，这时可以直接将该sst放在更高一层，尽可能的减少major compaction。

新生成的sst file以及其所在level都会记录在本次的versionEdit对象中，然后将该versionEdit应用在currentVersion生成新的version，最后将该versionEdit持久化到manifest文件。

- Major compaction

major compaction过程的细节比较多，作者的设计思想也是尽量做有必要的，尽可能优化性能，还需细细研究。

major compaction也可以手动触发，多数情况还是进行自动触发的。自动触发会根据平时的统计信息来决定什么时候对哪个level的哪个sst file进行comapction操作。

Level 0的sst file是最特殊的，sst file中的key range可以相互重叠，也就是给定一个key，这个key可能存在sst file中的任意一个或多个中，所以Level 0会做特殊处理。通过pickCompaction创建compaction任务，然后执行compaction。

LevelDB的简洁得益于它的抽象，比如多个文件的合并操作，在上层只需要创建多个文件的迭代器，就可以按key有序遍历数据，极大简化上层的调用。
