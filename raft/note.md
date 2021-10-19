# Raft library
大多数 Raft 实现都采用的是一个一体化的设计，包括存储的处理、消息的序列化和网络传输。而 etcd 的 raft lib 遵循最小化设计原则，仅实现了核心的 raft 算法。这种极简主义带来了 flexibility、determinism 和 performance。

由于该 lib 仅实现了 Raft 算法，用户需要自己完成网络和磁盘 IO。
- 用户必须实现自己的传输层，以便通过网络在 Raft peers 之间传递消息。
- 用户必须实现自己的存储层，以持久化 Raft 日志和状态。 


# Features

etcd raft implementation 提供的 Features 包括：
- Leader election
- Log replication
- Log compaction
- Membership changes
- Leadership transfer extension
- Efficient linearizable read-only queries served by both the leader and followers
  - leader checks with quorum and bypasses Raft log before processing read-only queries
  - followers asks leader to get a safe read index before processing read-only queries
- More efficient lease-based linearizable read-only queries served by both the leader and followers
  - leader bypasses Raft log and processing read-only queries locally
  - followers asks leader to get a safe read index before processing read-only queries
  - this approach relies on the clock of the all the machines in raft group

以及一些可选的 enhancements:
- Optimistic pipelining to reduce log replication latency
- Flow control for log replication
- Batching Raft messages to reduce synchronized network I/O calls
- Batching log entries to reduce disk synchronized I/O
- Writing to leader's disk in parallel
- Internal proposal redirection from followers to leader
- Automatic stepping down when the leader loses quorum
- Protection against unbounded log growth when quorum is lost

## Usage

The primary object in raft is a Node. Either start a Node from scratch using raft.StartNode or start a Node from some initial state using raft.RestartNode.

To start a three-node cluster
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

Start a single node cluster, like so:
```go
  // Create storage and config as shown above.
  // Set peer list to itself, so this node can become the leader of this single-node cluster.
  peers := []raft.Peer{{ID: 0x01}}
  n := raft.StartNode(c, peers)
```

To allow a new node to join this cluster, do not pass in any peers. 
1. First, add the node to the existing cluster by calling `ProposeConfChange` on any existing node inside the cluster.
2. Then, start the node with an empty peer list, like so:
```go
  // Create storage and config as shown above.
  n := raft.StartNode(c, nil)
```

To restart a node from previous state:
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

    // Restart raft without peer information.
    // Peer information is already included in the storage.
    n := raft.RestartNode(c)
```

建立 Node 后，user 需要负责一下的 **responsibilities**:

1. 首先，用户需要从 `Node.Ready()` channel 读取数据并处理其包含的 updates。这些步骤中，除了 step 2 中指出的部分，都可以并行地执行。
> 1. 按序将 Entries、HardState 和 Snapshot 写入到持久存储，即先写 Entries，然后是 HardState，Snapshot（如果非空）。如果存储支持 atomic writes，那么可以一起写入。
>       - 需要注意的是，当写一个 $Index = i$ 的 Entry 时，必须丢弃掉任何索引 $Index >= i$ 的之前就完成持久化的 Entries。
> 2. 将所有的 Messages 发送到在 `To` 字段中标明的节点。
>       - 需要注意的是，在 latest HardState 被持久化到磁盘，并且在所有之前的 `Ready()` batch 所写的所有 Entries 之前，不会发送任何 messages（在同一 batch 的 entries 被持久化的同时时，Messages 也可以被发送）【Raft-extended 中没有，应该是大论文】
>       - 为了减少 I/O 延迟，可以采用一种优化方法，使 leader 与其 followers 并行地写磁盘（Raft 论文中的 10.2.1节）。
>       - 如果任何消息的类型是 `MsgSnap`，在它被发送后，调用 `Node.ReportSnapshot()`（这些 messages 可能会很大）。
>       - 注意：Marshalling messages 不是线程安全的；要确保在 marshalling 时没有新的 entries 正被持久化。实现这一点的最简单方法是直接在 main raft loop 中序列化 messages。
> 3. 应用 Snapshot（如果有的话）和 CommittedEntries 到状态机中。
>       - 如果任何 committed Entry 是 `EntryConfChange` 类型，调用 `Node.ApplyConfChange()` 将其应用于该节点。
>       - 可以在调用 `ApplyConfChange()` 前将 NodeID 字段置 0 来取消 configuration change（但 `ApplyConfChange()` 必须被调用，而且取消 config change 的决定必须完全基于状态机，而不是外部的信息，如观察到的节点的健康状况）
> 4. 调用 `Node.Advance()` 以 singal 准备好进行下一个 update batch。
>       - 这一步可以在 step 1 之后的任何时间进行，但所有的 updates 必须按照 `Ready()` 返回的顺序进行处理。

2. 第二，所有持久化的 log entries 必须通过一个存储接口的实现来提供。可以使用 raft 提供的 MemoryStorage（如果在重启时重新填充其状态），或者可以提供一个自定义的以持久化存储作为后端支持的实现。

3. 第三，在收到另一个节点的消息后，将其传递给 `Node.Step`。
```go
	func recvRaftRPC(ctx context.Context, m raftpb.Message) {
		n.Step(ctx, m)
	}
```

4. 最后，定期调用 `Node.Tick()`。 Raft 有两个重要的 timeouts：heartbeat timeout 和 election timeout。在 raft package 内部，time 被抽象为 “tick”。

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

要通过接收应用数据的节点 propose changes 到状态机，需要序列化数据为一个 byte slice，并调用
```go
	n.Propose(ctx, data)
```

如果这个 proposal 被 committed，数据将出现在 committed entries，且数据的类型为 `raftpb.EntryNormal`。raft 不保证 proposed command 会 committed；该命令可能需要 timeout 后 repropose。

要在一个 cluster 中增/删 node，需要构造一个 ConfChange struct(如 cc)，并调用
```go
	n.ProposeConfChange(ctx, cc)
```

在 config change 被 committed 后，会返回一些类型为 `raftpb.EntryConfChange` 的 committed entry。这必须通过以下方式 apply 到节点：
```go
	var cc raftpb.ConfChange
	cc.Unmarshal(data)
	n.ApplyConfChange(cc)
```

注意：etcd-raft 中，ID 始终代表集群中的一个唯一节点。即使旧节点已被删除，给定的 ID 也不会被重复使用使用。并且 node ID 必须非零。


## Implementation notes

This implementation is up to date with the final Raft thesis (https://github.com/ongardie/dissertation/blob/master/stanford.pdf), although this implementation of the membership change protocol differs somewhat from that described in chapter 4. The key invariant that membership changes happen one node at a time is preserved, but in our implementation the membership change takes effect when its entry is applied, not when it is added to the log (so the entry is committed under the old membership instead of the new). This is equivalent in terms of safety, since the old and new configurations are guaranteed to overlap.

To ensure there is no attempt to commit two membership changes at once by matching log positions (which would be unsafe since they should have different quorum requirements), any proposed membership change is simply disallowed while any uncommitted change appears in the leader's log.

This approach introduces a problem when removing a member from a two-member cluster: If one of the members dies before the other one receives the commit of the confchange entry, then the member cannot be removed any more since the cluster cannot make progress. For this reason it is highly recommended to use three or more nodes in every cluster.

## Go docs

More detailed development documentation can be found in go docs: https://pkg.go.dev/go.etcd.io/etcd/raft/v3.