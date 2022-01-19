# Raft library
大多数 Raft 实现都采用的是一个一体化的设计, 包括存储的处理、消息的序列化和网络传输. 而 etcd 的 raft lib 遵循最小化设计原则, 仅实现了核心的 raft 算法. 这种极简主义带来了 flexibility、determinism 和 performance. 

由于该 lib 仅实现了 Raft 算法, 用户需要自己完成网络和磁盘 IO. 
- 用户必须实现自己的传输层, 以便通过网络在 Raft peers 之间传递消息. 
- 用户必须实现自己的存储层, 以持久化 Raft 日志和状态.  

为了方便测试 Raft lib, 其行为应该是确定性的。为了实现这种 determinism, 该库将 Raft 建模为一个状态机.  
- 状态机的输入是 `Message`.
  - Message 这一概念可以是 local timer update, 也可以是来自 remote peer 的网络消息.
- 状态机的输出是一个 3 元组 `{[]Messages, []LogEntries, NextState}`
  - `NextState` 是 Raft state changes

对于具有相同状态的状态机, 给予相同的 state machine input 应该总是得到相同的 state machine output.

# Features
etcd raft implementation 提供的 Features 包括:
- Leader election
- Log replication
- Log compaction
- Membership changes
- Leadership transfer extension
- 可由 leader 和 followers 提供的, 高效的 linearizable read-only queries
  - 在处理 read-only queries 之前, leader 会检查 quorum 并 bypass Raft log
  - 在处理 read-only queries 之前, followers 可以向 leader 获取一个 safe read index
- 可由 leader 和 followers 提供的, 更高效的 lease-based linearizable read-only queries
  - leader 可以 bypass Raft log, 直接 locally 的响应 read-only queries
  - 在处理 read-only queries 之前, followers 可以向 leader 获取一个 safe read index
  - 该方法依赖整个 raft group 中所有机器的物理时钟

以及一些可选的 enhancements:
- Optimistic pipelining 以减少 log replication 的延迟
- log replication 的流量控制
- Batching Raft messages 以减少 synchronized network I/O 调用
- Batching log entries 以减少 synchronized disk I/O
- 并行地写 leader 的磁盘
- followers -> leader 的 Internal proposal 重定向
- 当 leader loses quorum 时自动降级下台
- quorum lose 情况下防止 log 无限增长


## Usage
etcd raft lib 中的 primary object 是 `Node`. 使用 etcd raft 的入口或者是通过 `raft.StartNode` 从头开始, 或者是通过 `raft.RestartNode` 从某个初始状态开始.  

比如启动过一个 3 节点的 cluster
```go
    storage := raft.NewMemoryStorage()
    c := &raft.Config{
        ID:              0x01,
        ElectionTick:    10,
        HeartbeatTick:   1,
        Storage:         storage,
        MaxSizePerMsg:   4096,
        MaxInflightMsgs: 256,
    }
    // Set peer list to the other nodes in the cluster.
    // Note that they need to be started separately as well.
    n := raft.StartNode(c, []raft.Peer{{ID: 0x02}, {ID: 0x03}})
```

或者启动一个 1 节点的 cluster
```go
    // Create storage and config as shown above.
    // Set peer list to itself, so this node can become the leader of this single-node cluster.
    peers := []raft.Peer{{ID: 0x01}}
    n := raft.StartNode(c, peers)
```

当需要向一个 cluster 添加一个新的 Node 时, 应该：
1. 首先在该集群的任意一个已有 Node 上通过调用 `ProposeConfChange` 将一个节点添加到集群
2. 然后以一个 empty peer list 为参数启动这个节点
```go
    // Create storage and config as shown above.
    n := raft.StartNode(c, nil)
```

如果是需要从一个 previous state 重启节点:
```go
    storage := raft.NewMemoryStorage()

    // Recover the in-memory storage from persistent snapshot, state and entries.
    storage.ApplySnapshot(snapshot)
    storage.SetHardState(state)
    storage.Append(entries)

    c := &raft.Config{
        ID:              0x01,
        ElectionTick:    10,
        HeartbeatTick:   1,
        Storage:         storage,
        MaxSizePerMsg:   4096,
        MaxInflightMsgs: 256,
    }

    // Restart raft 时不许要 peer 信息, 这些信息已经包含在了 storage 中
    n := raft.RestartNode(c)
```

创建好 Node 之后, user 需要负责以下的 **responsibilities**:

**第一**, 用户需要从 `Node.Ready()` channel 读取数据并处理其包含的 updates. 这些步骤中, 除了 **step 2** 中指出的部分, 其他都可以并行地执行. 
1. 按序将 `Entries`、`HardState` 和 `Snapshot` 写入到持久化存储. 即先写 Entries, 然后是 HardState, Snapshot（如果非空）. 
   - 如果存储支持 atomic writes, 那么可以一起写入. 
   - 需要注意的是, 当写一个 $Index = i$ 的 Entry 时, 必须丢弃掉任何之前就完成持久化 $Index >= i$ 的 Entries. 
2. 将所有的 Messages 发送到在 `To` 字段中标明的节点. 
   - 需要注意的是, 在 latest HardState 被持久化到磁盘, 并且在所有之前的 `Ready()` batch 所写的所有 Entries 之前, 不应发送任何 messages（在同一 batch 的 entries 被持久化的同时时, Messages 也可以被发送）【Raft-extended 中没有, 应该是大论文】
   - 为了减少 I/O 延迟, 可以采用一种优化方法, 使 leader 与其 followers 并行地写磁盘（Raft 论文中的 10.2.1节）. 
   - 如果一个 Message 的类型是 `MsgSnap`, 在它被发送后, 调用 `Node.ReportSnapshot()`（这些 messages 可能会很大）. 
   - 注意：Marshalling messages 不是线程安全的；要确保在 marshalling 时没有新的 entries 正被持久化.
     - 实现这一点的最简单方法是直接在 main raft loop 中序列化 messages. 
3. 应用 Snapshot（如果有的话）和 CommittedEntries 到状态机中. 
   - 如果任何 committed Entry 是 `EntryConfChange` 类型, 调用 `Node.ApplyConfChange()` 将其应用于该节点. 
     - 可以在调用 `ApplyConfChange()` 前将 NodeID 字段置 0 来取消 configuration change（但 `ApplyConfChange()` 必须被调用, 而且取消 config change 的决定必须完全基于状态机, 而不是任何外部的信息, 如观察到的节点的健康状况）
4. 调用 `Node.Advance()` 以表明准备好进行下一个 update batch. 
   - 这一步可以在 step 1 之后的任何时刻进行, 但所有的 updates 必须按照 `Ready()` 返回的顺序进行处理. 

**第二**, 所有 persisted log entries 必须通过一个 `Storage` 接口的实现来提供. 可以使用 raft 提供的 MemoryStorage（如果在重启时重新填充其状态）, 或者可以提供一个自定义的以持久化存储作为后端的实现. 

**第三**, 在收到另一个节点的 message 后, 将其传递给 `Node.Step`. 
```go
	func recvRaftRPC(ctx context.Context, m raftpb.Message) {
		n.Step(ctx, m)
	}
```

**第四**, 定期调用 `Node.Tick()`. Raft 有两个重要的 timeouts：heartbeat timeout 和 election timeout. 在 raft package 内部, time 被抽象为 “tick”. 

整个状态机的 handling loop 大致如下：
```go
for {
    select {
    case <-s.Ticker:
        n.Tick()
    case rd := <-s.Node.Ready():
        saveToStorage(rd.HardState, rd.Entries, rd.Snapshot)
        send(rd.Messages)
        if !raft.IsEmptySnap(rd.Snapshot) {
            processSnapshot(rd.Snapshot)
        }
        for _, entry := range rd.CommittedEntries {
            process(entry)
            if entry.Type == raftpb.EntryConfChange {
                var cc raftpb.ConfChange
                cc.Unmarshal(entry.Data)
                s.Node.ApplyConfChange(cc)
            }
        }
        s.Node.Advance()
    case <-s.done:
        return
    }
}
```

要通过接收应用数据的节点 propose changes 到状态机, 需要将数据序列化为一个 byte slice, 并调用
```go
	n.Propose(ctx, data)
```

如果这个 proposal 被 committed, `data` 将出现在 committed entries 中, 且类型为 `raftpb.EntryNormal`. raft 不保证 proposed command 会 committed; 该 command 可能需要在 timeout 后 re-propose. 

要在一个 cluster 中增/删 node, 需要构造一个 ConfChange struct(如 cc), 并调用
```go
	n.ProposeConfChange(ctx, cc)
```

在 config change 被 committed 后, 会返回一些类型为 `raftpb.EntryConfChange` 的 committed entry. 必须通过以下方式将其 apply 到节点：
```go
	var cc raftpb.ConfChange
	cc.Unmarshal(data)
	n.ApplyConfChange(cc)
```

注意：etcd-raft 中, ID 始终代表集群中的一个唯一节点. 即使旧节点已被删除, 给定的 ID 也不会被重复使用使用. 并且 node ID 必须非零. 


## Implementation notes

etcd raft implementation 与 [final Raft thesis](https://github.com/ongardie/dissertation/blob/master/stanford.pdf) 是同步的, 但在 membership change protocol 上的实现与论文中 chapter 4 所描述的有些不同. 我们保留了 membership changes 每次只会发生在一个节点的关键不变性, 但在 etcd raft 中, membership change 在其 entry 被应用时生效, 而不是在它被添加到 log 时生效 (即该 entry 是在 old membership 下 committed, 而不是 new membership). 这在安全性上等价的，因为新旧配置被保证会有 overlap.

为了确保不会尝试通过 matching log positions 来同时提交两个 membership changes (这是不安全的，因为这两次 membership change 应该有着不同的 quorum requirements), 当 leader 的 log 中存在 uncommitted change 时, 我们简单地禁止任何 proposed membership change.

这种方法在从一个 two-member 集群中移除一个成员时是有问题的. 如果其中成员 A 在成员 B 收到 confchange entry 的 commit 之前死亡, 那么成员 A 就不能被移除, 因为此时的集群无法推动状态机前进. 因此，我们强烈建议在每个集群中使用三个或更多的节点.


## MessageType
Package raft 以 pb 格式 (定义于 raftpb package) 发送和接收消息. 每个 state (follower, candidate, leader) 对于给定的 `raftpb.Message` 进行 advancing 时都实现自己的 'step' method (`stepFollower`、`stepCandidate`、`stepLeader`). 每个 step 由其 `raftpb.MessageType` 决定. 请注意，每个 step 都会由一个共同的方法 `Step` 来检查, 该方法对 node 当前的 term 和 incoming message 的 term 进行安全检查，以防止 stale log entries:

- `MsgHup` 用于 election. 如果一个节点是 follower 或 candidate, 则 'raft' struct 的 `tick` 函数被设置为 `tickElection`. 如果 follower 或 candidate 在 election timeout 前没有收到任何心跳，则会向其 `Step` method 传入一个 `MsgHup`, 并成为（或保持） candidate 身份, 开始一轮新的选举.

- `MsgBeat` 是一个 internal type, 用于通知 leader 发送 `MsgHeartbeat` 类型的心跳. 如果一个节点是 leader, 则其 'raft' struct 的 `tick` 函数被置为 `tickHeartbeat`, 并触发其向 followers 发送周期性的 `MsgHeartbeat` 消息.

- `MsgProp` 用于 propose 将 data 给 append 到其 log entries 中. 这是一个特殊的类型，用于将 proposals 重定向到 leader. 因此, `send` 方法会用其 `HardState` 中的 term 覆盖 `raftpb.Message` 的 term，以避免将其 local term 附加到 `MsgProp`. 
  - 当 `MsgProp` 被传递给 leader 的 `Step` 方法时, leader 首先调用 `appendEntry` 方法将 entries 给 append 到其日志中，然后调用 `bcastAppend` 方法将这些 entries 发送给其 peers. 
  - 当 `MsgProp` 被传递给 candidate 时, `MsgProp` 将被丢弃.
  - 当 `MsgProp` 被传递给 follower 时, `MsgProp` 通过 send 方法存储在 follower 的 mailbox(msgs) 中. `MsgProp` 与 sender's ID 一起被存储起来, 并在之后通过 rafthttp package 转发给 leader.

- `MsgApp` 中包含要复制的 log entries. 当 leader 调用 `bcastAppend`, `bcastAppend` 会调用 `sendAppend`, 以 `MsgApp` 类型发送 soon-to-be-replicated logs. 当 `MsgApp` 被传递给一个 candidate 的 `Step` 方法时, candidate 会重新回退成 follower, 因为这条消息表明当前有一个 valid leader. Candidate 和 follower 会以 `MsgAppResp` 回应 `MsgApp`.

- `MsgAppResp` 用于对 log replication request(`MsgApp`) 作出响应. 当 `MsgApp` 被传递给 candidate 或 follower 的 `Step` 方法时, 接收者会通过调用 `handleAppendEntries` 方法进行响应, 该方法会发送 `MsgAppResp` 到 raft mailbox.

- `MsgVote` 用于选举中请求 votes. 当一个节点是 follower 或 candidate, 并且有 `MsgHup` 传入其 `Step` 方法, 此时该节点就会调用 campaign 方法来竞选成为一个 leader. 一旦 `campaign` 方法被调用, 该节点就会成为 candidate, 并向集群中的 peers 发送 `MsgVote` 以请求投票.
  - 当一个 `MsgVote` 被传递给 leader 或 candidate 的 `Step` 方法时.
    - 如果 `MsgVote` 的 Term 小于 leader 或 candidate 的 Term, 则该 `MsgVote` 将被拒绝 (返回一个 Reject 字段为 true 的 `MsgVoteResp`).
    - 如果 `MsgVote` 的 Term 大于 leader 或 candidate 的 Term, 那么它们将回退为 follower 状态.
  - 当一个 `MsgVote` 被传递给 follower 时, 只有满足如下的条件之一时才会投票给 sender.
    - 只有当 sender 的 last term 大于 `MsgVote` 的 term
    - 或者 sender 的 last term 等于 `MsgVote` 的 term，但 sender 的 last committed index 大于等于 follower 的 index

- `MsgVoteResp` 用于表示对于 voting request 的响应. 当 `MsgVoteResp` 被传递给 candidate 时，该 candidate 会计算自己获得了多少选票. 
  - 如果其选票超过了 majority (quorum), 则成为 leader 并调用 `bcastAppend`. 
  - 如果收到的拒绝票占 majority, 则该节点会回退为 follower.

- `MsgPreVote` 和 `MsgPreVoteResp` 用于一种可选的 two-phase election protocol. 当 Config.PreVote 为 true 时, 则首先会进行 pre-election (使用与常规 election 相同的规则). 只有 pre-election 表明该竞选节点可以胜选的情况下, 该节点才会递增其 term.
  - 该机制用于最大限度地降低一个 partitioned node 在重新加入集群时造成的干扰.

- `MsgSnap` 用于要求 install 一个 snapshot. 当一个节点刚刚成为 leader, 或者当 leader 收到了 `MsgProp` 消息时, 它会调用 `bcastAppend` 方法, 进而调用 `sendAppend` 方法给每个 follower. 在 `sendAppend` 中，如果 leader 未能获得 term 或 entries, leader 会通过发送 `MsgSnap` 消息请求快照.

- `MsgSnapStatus` 用于告知 snapshot install message 的结果. 当一个 follower 拒绝了 `MsgSnap` 时, 它表明 `MsgSnap` 中的快照请求由于网络问题, 无法向其 followers 发送快照而失败. leader 将追随者的 progress 视为探针. 当 `MsgSnap` 没有被拒绝时, 它表明快照成功, 并且 leader 会 follower's progress 置为 probe, 并恢复其 log replication.

- `MsgHeartbeat` 用于从 leader 出发送心跳. 当 `MsgHeartbeat` 被传递给 candidate, 且 `MsgHeartbeat` 中的 term 高于 candidate 的 term 时, candidate 就会回退为 follower, 并根据这个心跳中的信息更新其 committed index. 然后它会将 `MsgHeartbeat` 发送到它的 mailbox. 当 `MsgHeartbeat` 被传递给 follower 的 Step 方法时, 且 `MsgHeartbeat` 的 term 高于 follower 的 term, 则 follower 会用 `MsgHeartbeat` 中的 ID 更新其 leaderID.
  
- `MsgHeartbeatResp` 用于对 `MsgHeartbeat` 作出回应. 当 `MsgHeartbeatResp` 被传递给 leader 的 `Step` 方法时, leader 则知道了哪个 follower 已做出了回应. 只有当 leader 的 last committed index 大于 follower 的 Match index 时, leader 才会运行 `sendAppend` 方法。

- `MsgUnreachable` 表示 request(message) 没有被传递. 当 `MsgUnreachable` 被传递给 leader 的 `Step` 方法时, leader 会得知这个 `MsgUnreachable` 对应的 follower 不可达, 通常表明 `MsgApp` 丢失. 当 follower's progress state 是 replicate 时, leader 会将其置为 probe.


## Go docs

More detailed development documentation can be found in go docs: https://pkg.go.dev/go.etcd.io/etcd/raft/v3.