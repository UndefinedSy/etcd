## Progress

Progress 代表 leader 视角下的一个 follower 的进度。Leader 会维护所有 followers 的进度，并根据其进度向 follower 发送 `replication message`。

`replication message` 是一个带有 log entries 的 `msgApp`。

Progress 有两个属性：`match` 和 `next`。
- `match` 是已的匹配的下标值最高条目的索引。如果 leader 不知道一个 follower 的 replication status，则将 `match` 设置为 0。
- `next` 是将被复制给一个 follow 的下一个 entry 的索引。
  - leader 会将从 `next` 到其最新的所有 log entries 放到下一个 `replication message`

状态机变化图如下：
```
                            +--------------------------------------------------------+          
                            |                  send snapshot                         |          
                            |                                                        v          
                  +---------+----------+                                  +--------------------+
              +-->|       probe        |                                  |      snapshot      |
              |   |  max inflight = 1  |<---------------------------------+  max inflight = 0  |
              |   +---------+----------+                                  +--------------------+
              |             |            1. snapshot success                                    
              |             |               (next = snapshot.index + 1)                           
              |             |            2. snapshot failure                                    
              |             |               (no change)                                         
              |             |            3. receives msgAppResp(rej = false && index > lastsnap.index)
              |             |               (match = m.index, next = match+1)                        
receives msgAppResp(rej=true)                                                                   
(next=match+1)|             |                                                                   
              |             |                                                                   
              |             |                                                                   
              |             |   receives msgAppResp(rej=false&&index>match)                     
              |             |   (match=m.index,next=match+1)                                    
              |             |                                                                   
              |             |                                                                   
              |             v                                                                   
              |   +--------------------+                                                        
              |   |     replicate      |                                                        
              +---+  max inflight = n  |                                                        
                  +--------------------+                                                        
```
- 当一个 follower 的进度处于 `probe` 状态时，leader 在每个心跳间隔内最多发送一条 `replication message`。leader 会缓慢地发送 `replication message`，并探测 follower 的实际进度。一条表示 reject 的 `msgHeartbeatResp` 或 `msgAppResp` 可能会触发下一个`replication message` 的发送。  
When the progress of a follower is in `probe` state, leader sends at most one `replication message` per heartbeat interval. The leader sends `replication message` slowly and probing the actual progress of the follower. A `msgHeartbeatResp` or a `msgAppResp` with reject might trigger the sending of the next `replication message`.

- 当一个 follower 的进度处于 `replicate` 状态时，leader 会发送 `replication message`，然后乐观地将 `next` 增加到最新发送的条目。
    > 这是一个 optimized state，用于向 follower 进行快速复制日志条目。

- 当一个 follower 的进度处于 `snapshot` 状态时，leader 将停止发送任何 `replication message`。


当一个新的 leader 当选时，其会将所有 followers 的进度设置为: `state = probe`，`match = 0`，`next = last index`。然后 leader 会缓慢地（最多每次心跳一次的频率）向 follower 发送 `replication message`，并探测其进度。

当一个 follower 的 `msgAppResp` 不再是 rejection 时，就意味着它已经追上了发送的 index，此时 leader 可以将其进度变更为 `replicate`。此时，leader 开始向 follower 快速地流式地传输 log entries。直到 follower 回复一个拒绝的 `msgAppResp`，或发现 follower 不可达时，其进度将回退到 `probe`。

etcd-raft 中会主动地将 `next` reset 为 `match` + 1，因为如果我们很快就能收到任何 `msgAppResp`，`match` 和 `next` 将直接增加到 `msgAppResp` 的 `index`。(当主动 reset 的 `next` 过低时，我们可能最终会发送一些重复的 log entries，详见 open question)

当一个 follower 的进度非常滞后并需要快照时，其进度会从 `probe` 回退为 `snapshot`。在发送 `msgSnap` 后，领导者等待发送的快照的成功、失败或 abortion。当发送的快照被 applied 后，进度将回到 `probe`。

### Flow Control
1. 限制了每条 message 的最大发送量（可配置的），由于我们限制了每条消息的大小，因此降低了探测状态的成本；也降低了当我们主动下降到一个 too low 的 `next` 时的 penalty。

2. 在 `replicate` 状态下，限制 in flight 的信息数量 < N（N 是可配置的）。大多数实现会在其实际的网络传输层之上有一个 sending buffer（不用 blocking raft node）。我们要确保 raft 不会溢出该 buffer，否则可能会导致消息被丢弃，并引发一堆不必要的重复发送。