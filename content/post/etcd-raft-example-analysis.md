---
title: etcd raft lib使用示例分析
categories: [技术记录]
date: 2019-09-23 21:00:38
---


此前使用hashicorp raft库简单实现了一个redis协议的kv存储[Leto](https://github.com/yongman/leto)，[介绍文章](https://xiking.win/2018/07/30/implement-key-value-store-using-raft/)。

当时选择hashicorp raft库，包装好，对用户暴露的细节较少，上手使用简单；而看了etc的raft库的实例，对外暴露的细节较多，多个channel交互，只提供核心的raft算法，其余的操作需要自己动手实现，包括网络传输、存储等，在使用上也更方便需求定制实现。

看到很多raft库的工程实现或者库的引用，选择etcd raft库的占多数。比如k8s，tikv等。

#### 如何使用etcd raft库

raftexample就是一个最简单的使用示例，它实现的是一个简单的内存kv存储，通过HTTP API访问。

1. 初始化

由于raft库只实现核心逻辑，所以需要与应用进行交互。其交互采用4个channel：

- proposeC：用户代码向raft提交写请求Propose
- confChangeC：用户代码向raft提交配置变更请求ProposeConfChange
- commitC：用于raft向用户代码传输已经提交的entries，用户raft库暴露具体业务逻辑，将数据写入`State Machine`
- errorC：向用户代码暴露raft库的处理异常

其中proposeC和confChangeC是在开始手动创建，作为参数传递给raft组件。

commitC和errorC是用户代码手动封装raftNode时创建的channel，并在raft组件中使用。

```go
func newRaftNode(id int, peers []string, join bool, getSnapshot func() ([]byte, error), proposeC <-chan string,
	confChangeC <-chan raftpb.ConfChange) (<-chan *string, <-chan error, <-chan *snap.Snapshotter) {

	commitC := make(chan *string)
	errorC := make(chan error)

	rc := &raftNode{
		proposeC:    proposeC,
		confChangeC: confChangeC,
		commitC:     commitC,
		errorC:      errorC,
		id:          id,
		peers:       peers,
		join:        join,
		waldir:      fmt.Sprintf("raftexample-%d", id),
		snapdir:     fmt.Sprintf("raftexample-%d-snap", id),
		getSnapshot: getSnapshot,
		snapCount:   defaultSnapshotCount,
		stopc:       make(chan struct{}),

		snapshotterReady: make(chan *snap.Snapshotter, 1),
	}
	go rc.startRaft()
	return commitC, errorC, rc.snapshotterReady
}
```

2. 启动raftNode

在初始化的时候通过`rc.startRaft()`启动raft组件。

```go
func (rc *raftNode) startRaft() {
	rc.snapshotter = snap.New(zap.NewExample(), rc.snapdir)
	rc.snapshotterReady <- rc.snapshotter

	oldwal := wal.Exist(rc.waldir)
	rc.wal = rc.replayWAL()

	rpeers := make([]raft.Peer, len(rc.peers))
	for i := range rpeers {
		rpeers[i] = raft.Peer{ID: uint64(i + 1)}
	}
	c := &raft.Config{
		ID:                        uint64(rc.id),
		ElectionTick:              10,
		HeartbeatTick:             1,
		Storage:                   rc.raftStorage,
		MaxSizePerMsg:             1024 * 1024,
		MaxInflightMsgs:           256,
		MaxUncommittedEntriesSize: 1 << 30,
	}

	if oldwal {
		rc.node = raft.RestartNode(c)
	} else {
		startPeers := rpeers
		if rc.join {
			startPeers = nil
		}
		rc.node = raft.StartNode(c, startPeers)
	}

	rc.transport = &rafthttp.Transport{
		Logger:      zap.NewExample(),
		ID:          types.ID(rc.id),
		ClusterID:   0x1000,
		Raft:        rc,
		ServerStats: stats.NewServerStats("", ""),
		LeaderStats: stats.NewLeaderStats(strconv.Itoa(rc.id)),
		ErrorC:      make(chan error),
	}

	rc.transport.Start()
	for i := range rc.peers {
		if i+1 != rc.id {
			rc.transport.AddPeer(types.ID(i+1), []string{rc.peers[i]})
		}
	}

	go rc.serveRaft()
	go rc.serveChannels()
}
```

启动raft组件、创建网络传输、开始监听连接、监听channel数据。

其中serverChannel完成了核心逻辑，读取用户提交数据，提交给raft node处理，并且将entries持久化、同步给peer节点，然后将已经commit成功的entries写入commitC。

```go
func (rc *raftNode) serveChannels() {
	snap, err := rc.raftStorage.Snapshot()
	if err != nil {
		panic(err)
	}
	rc.confState = snap.Metadata.ConfState
	rc.snapshotIndex = snap.Metadata.Index
	rc.appliedIndex = snap.Metadata.Index

	defer rc.wal.Close()

	ticker := time.NewTicker(100 * time.Millisecond)
	defer ticker.Stop()

	// send proposals over raft
	go func() {
		confChangeCount := uint64(0)

		for rc.proposeC != nil && rc.confChangeC != nil {
			select {
			case prop, ok := <-rc.proposeC:
				if !ok {
					rc.proposeC = nil
				} else {
					// blocks until accepted by raft state machine
					rc.node.Propose(context.TODO(), []byte(prop))
				}

			case cc, ok := <-rc.confChangeC:
				if !ok {
					rc.confChangeC = nil
				} else {
					confChangeCount++
					cc.ID = confChangeCount
					rc.node.ProposeConfChange(context.TODO(), cc)
				}
			}
		}
		// client closed channel; shutdown raft if not already
		close(rc.stopc)
	}()

	// event loop on raft state machine updates
	for {
		select {
		case <-ticker.C:
			rc.node.Tick()

		// store raft entries to wal, then publish over commit channel
		case rd := <-rc.node.Ready():
			rc.wal.Save(rd.HardState, rd.Entries)
...
			rc.raftStorage.Append(rd.Entries)
			rc.transport.Send(rd.Messages)
			if ok := rc.publishEntries(rc.entriesToApply(rd.CommittedEntries)); !ok {
				rc.stop()
				return
			}
			rc.maybeTriggerSnapshot()
			rc.node.Advance()

		case err := <-rc.transport.ErrorC:
			rc.writeError(err)
			return

		case <-rc.stopc:
			rc.stop()
			return
		}
	}
}
```

通过监听commitC中的数据，最后完成数据写入状态机。

3. 数据写入

整个的数据写入过程：

- http api请求写入proposeC，监听到proposeC中有数据，将数据提交给raft
- 当raft处理完成后通过Ready chan将请求交给用户代码继续处理，完成wal、entries持久化和entries发送给peer，查找是否有已经commit的entries，将commit的entries写入commitC
- 用户代码通过从commitC中读取已经提交成功的数据，可以放心的将数据写入状态机。

raftexample中kvstore的具体写入逻辑：

```go
func (s *kvstore) readCommits(commitC <-chan *string, errorC <-chan error) {
	for data := range commitC {
		...
		var dataKv kv
		dec := gob.NewDecoder(bytes.NewBufferString(*data))
		if err := dec.Decode(&dataKv); err != nil {
			log.Fatalf("raftexample: could not decode message (%v)", err)
		}
		s.mu.Lock()
		s.kvStore[dataKv.Key] = dataKv.Val
		s.mu.Unlock()
	}
	if err, ok := <-errorC; ok {
		log.Fatal(err)
	}
}
```

至此完成数据写入。这里只是etcd raft库的简单使用，后续会深入阅读raft库的实现。



**参考**

1. https://xiking.win/2018/07/30/implement-key-value-store-using-raft/
2. https://github.com/etcd-io/etcd/tree/master/contrib/raftexample
3. https://studygolang.com/articles/10287