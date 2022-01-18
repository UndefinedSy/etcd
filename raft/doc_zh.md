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
包raft以协议缓冲区格式（在raftpb包中定义）发送和接收消息。每个状态（追随者、候选者、领导者）在用给定的raftpb.Message推进时实现它自己的'步骤'方法（'stepFollower'、'stepCandidate'、'stepLeader'）。每个步骤由其raftpb.MessageType决定。请注意，每个步骤都由一个共同的方法'Step'来检查，该方法对节点和传入消息的条件进行安全检查，以防止陈旧的日志条目。

Package raft sends and receives message in Protocol Buffer format (defined in raftpb package). Each state (follower, candidate, leader) implements its own 'step' method ('stepFollower', 'stepCandidate', 'stepLeader') when advancing with the given raftpb.Message. Each step is determined by its raftpb.MessageType. Note that every step is checked by one common method 'Step' that safety-checks the terms of node and incoming message to prevent stale log entries:

- 'MsgHup'是用来选举的。如果一个节点是追随者或候选人，'raft'结构中的'tick'函数被设置为'tickElection'。如果追随者或候选人在选举超时前没有收到任何心跳，它将'MsgHup'传递给它的Step方法，并成为（或保持）一个候选人，开始新的选举。

- 'MsgHup' is used for election. If a node is a follower or candidate, the 'tick' function in 'raft' struct is set as 'tickElection'. If a follower or candidate has not received any heartbeat before the election timeout, it passes 'MsgHup' to its Step method and becomes (or remains) a candidate to start a new election.

- MsgBeat'是一个内部类型，它向领导者发出信号，发送'MsgHeartbeat'类型的心跳声。如果一个节点是领导者，'raft'结构中的'tick'函数被设置为'tickHeartbeat'，并触发领导者向其追随者定期发送'MsgHeartbeat'消息。

- 'MsgBeat' is an internal type that signals the leader to send a heartbeat of the 'MsgHeartbeat' type. If a node is a leader, the 'tick' function in the 'raft' struct is set as 'tickHeartbeat', and triggers the leader to send periodic 'MsgHeartbeat' messages to its followers.

- 'MsgProp'提议将数据附加到其日志条目中。这是一个特殊的类型，用于重定向到领导者的提议。因此，发送方法用它的HardState的术语覆盖了raftpb.Message的术语，以避免将它的本地术语附加到'MsgProp'。当'MsgProp'被传递给领导者的'Step'方法时，领导者首先调用'appendEntry'方法将条目追加到其日志中，然后调用'bcastAppend'方法将这些条目发送给其同伴。当传递给候选者时，'MsgProp'被放弃。当传递给follower时，'MsgProp'通过send方法存储在follower的邮箱（msgs）中。它与发件人的ID一起被存储起来，并在以后由rafthttp包转发给领导者。

- 'MsgProp' proposes to append data to its log entries. This is a special type to redirect proposals to leader. Therefore, send method overwrites raftpb.Message's term with its HardState's term to avoid attaching its local term to 'MsgProp'. When 'MsgProp' is passed to the leader's 'Step' method, the leader first calls the 'appendEntry' method to append entries to its log, and then calls 'bcastAppend' method to send those entries to its peers. When passed to candidate, 'MsgProp' is dropped. When passed to follower, 'MsgProp' is stored in follower's mailbox(msgs) by the send method. It is stored with sender's ID and later forwarded to leader by rafthttp package.

- MsgApp "包含要复制的日志条目。一个领导者调用bcastAppend，后者调用sendAppend，后者以'MsgApp'类型发送即将被复制的日志。当'MsgApp'被传递到候选者的Step方法中时，候选者会重新回到跟随者，因为它表明有一个有效的领导者在发送'MsgApp'消息。候选者和跟随者以'MsgAppResp'类型回应这个消息。
  
- 'MsgApp' contains log entries to replicate. A leader calls bcastAppend, which calls sendAppend, which sends soon-to-be-replicated logs in 'MsgApp' type. When 'MsgApp' is passed to candidate's Step method, candidate reverts back to follower, because it indicates that there is a valid leader sending 'MsgApp' messages. Candidate and follower respond to this message in 'MsgAppResp' type.

- MsgAppResp'是对日志复制请求（'MsgApp'）的响应。当'MsgApp'被传递给候选人或追随者的步骤方法时，它通过调用'handleAppendEntries'方法进行响应，该方法将'MsgAppResp'发送到raft邮箱。
  
- 'MsgAppResp' is response to log replication request('MsgApp'). When 'MsgApp' is passed to candidate or follower's Step method, it responds by calling 'handleAppendEntries' method, which sends 'MsgAppResp' to raft mailbox.

- 'MsgVote'请求选举投票。当一个节点是追随者或候选人，并且'MsgHup'被传递到它的Step方法，那么该节点就会调用'campaign'方法来竞选自己成为一个领导者。一旦'竞选'方法被调用，该节点就成为候选人，并向集群中的同伴发送'MsgVote'以请求投票。当传递给领导者或候选人的步骤方法时，如果消息的期限低于领导者或候选人的期限，'MsgVote'将被拒绝（'MsgVoteResp'将以Reject true返回）。如果领导者或候选人收到的'MsgVote'的术语更高，那么它将返回给跟随者。当'MsgVote'被传递给follower时，只有当sender的最后期限大于MsgVote的期限，或者sender的最后期限等于MsgVote的期限，但sender最后承诺的索引大于或等于follower的索引，它才会投票给sender。

- 'MsgVote' requests votes for election. When a node is a follower or candidate and 'MsgHup' is passed to its Step method, then the node calls 'campaign' method to campaign itself to become a leader. Once 'campaign' method is called, the node becomes candidate and sends 'MsgVote' to peers in cluster to request votes. When passed to leader or candidate's Step method and the message's Term is lower than leader's or candidate's, 'MsgVote' will be rejected ('MsgVoteResp' is returned with Reject true). If leader or candidate receives 'MsgVote' with higher term, it will revert back to follower. When 'MsgVote' is passed to follower, it votes for the sender only when sender's last term is greater than MsgVote's term or sender's last term is equal to MsgVote's term but sender's last committed index is greater than or equal to follower's.

- MsgVoteResp'包含来自投票请求的响应。当'MsgVoteResp'被传递给候选人时，候选人会计算自己赢得了多少票。如果它超过了多数票（法定人数），它就成为领导者，并调用'bcastAppend'。如果候选者收到了多数的拒绝票，它就会变回追随者。

- 'MsgVoteResp' contains responses from voting request. When 'MsgVoteResp' is passed to candidate, the candidate calculates how many votes it has won. If it's more than majority (quorum), it becomes leader and calls 'bcastAppend'. If candidate receives majority of votes of denials, it reverts back to follower.

- MsgPreVote'和'MsgPreVoteResp'用于一个可选的两阶段选举协议。当Config.PreVote为真时，首先进行预选（使用与常规选举相同的规则），除非预选表明竞选节点将获胜，否则没有节点增加其任期数。这最大限度地减少了被分区的节点重新加入集群时的干扰。
  
- 'MsgPreVote' and 'MsgPreVoteResp' are used in an optional two-phase election protocol. When Config.PreVote is true, a pre-election is carried out first (using the same rules as a regular election), and no node increases its term number unless the pre-election indicates that the campaigning node would win. This minimizes disruption when a partitioned node rejoins the cluster.

- 'MsgSnap'请求安装一个快照信息。当一个节点刚刚成为领导者或者领导者收到'MsgProp'消息时，它会调用'bcastAppend'方法，然后调用'sendAppend'方法给每个跟随者。在'sendAppend'中，如果领导者未能获得术语或条目，领导者通过发送'MsgSnap'类型的消息请求快照。

- 'MsgSnap' requests to install a snapshot message. When a node has just become a leader or the leader receives 'MsgProp' message, it calls 'bcastAppend' method, which then calls 'sendAppend' method to each follower. In 'sendAppend', if a leader fails to get term or entries, the leader requests snapshot by sending 'MsgSnap' type message.

- MsgSnapStatus'告诉了快照安装消息的结果。当追随者拒绝了'MsgSnap'时，它表明'MsgSnap'的快照请求因网络问题而失败，这导致网络层无法向其追随者发送快照。然后领导者将追随者的进度视为探测。当'MsgSnap'没有被拒绝时，它表明快照成功了，领导者将跟随者的进度设置为探测，并恢复其日志复制。

- 'MsgSnapStatus' tells the result of snapshot install message. When a follower rejected 'MsgSnap', it indicates the snapshot request with 'MsgSnap' had failed from network issues which causes the network layer to fail to send out snapshots to its followers. Then leader considers follower's progress as probe. When 'MsgSnap' were not rejected, it indicates that the snapshot succeeded and the leader sets follower's progress to probe and resumes its log replication.

- MsgHeartbeat'从领导者那里发送心跳信号。当'MsgHeartbeat'被传递给候选者，并且消息的期限高于候选者的期限时，候选者就会返回到跟随者，并从这个心跳中更新它的承诺索引。然后它将消息发送到它的邮箱。当'MsgHeartbeat'被传递到follower的Step方法中，并且消息的期限高于follower的，follower就会用消息中的ID更新它的leaderID。

- 'MsgHeartbeat' sends heartbeat from leader. When 'MsgHeartbeat' is passed to candidate and message's term is higher than candidate's, the candidate reverts back to follower and updates its committed index from the one in this heartbeat. And it sends the message to its mailbox. When 'MsgHeartbeat' is passed to follower's Step method and message's term is higher than follower's, the follower updates its leaderID with the ID from the message.

- MsgHeartbeatResp'是对'MsgHeartbeat'的一个回应。当'MsgHeartbeatResp'被传递给领导者的Step方法时，领导者知道哪个追随者做出了回应。只有当领导者最后提交的索引大于跟随者的匹配索引时，领导者才会运行'sendAppend'方法。
  
- 'MsgHeartbeatResp' is a response to 'MsgHeartbeat'. When 'MsgHeartbeatResp' is passed to leader's Step method, the leader knows which follower responded. And only when the leader's last committed index is greater than follower's Match index, the leader runs 'sendAppend` method.

- MsgUnreachable "表示请求（消息）没有被传递。当'MsgUnreachable'被传递给领导者的Step方法时，领导者发现发送这个'MsgUnreachable'的跟随者无法到达，通常表明'MsgApp'丢失。当跟随者的进度状态被复制时，领导者会将其设置为探测。
  
- 'MsgUnreachable' tells that request(message) wasn't delivered. When 'MsgUnreachable' is passed to leader's Step method, the leader discovers that the follower that sent this 'MsgUnreachable' is not reachable, often indicating 'MsgApp' is lost. When follower's progress state is replicate, the leader sets it back to probe.



## Go docs

More detailed development documentation can be found in go docs: https://pkg.go.dev/go.etcd.io/etcd/raft/v3.