## gRPC Services
etcd3 RPCs 根据功能分成了若干个 services。

其中处理 key space 的服务包括:
- KV: 对 kv pairs 的 creates, updates, fetches, deletes
- Watch: Monitors changes to keys
- Lease: 用于处理 client 的 keep-alive messages 的原语

其中用于管理集群的服务包括:
- Auth: role-based 的身份验证机制，用于对用户进行身份验证。
- Cluster: 提供成员信息和 configuration 工具。
- Maintenance: 
  - 获取 recovery snapshots
  - 对存储进行碎片整理
  - 返回每个成员的状态信息。


### Requests and Responses

所有 etcd API 的 response 都会附带上一个 response header，其中包含了该 response 的 cluster metadata:
```go
message ResponseHeader {
    uint64 cluster_id = 1;
    uint64 member_id  = 2;
    int64  revision   = 3;
    uint64 raft_term  = 4;
}
```
- Cluster_ID: 生成该 response 的 cluster 的 ID.
- Member_ID:  生成该 response 的 member 的 ID.
- Revision:   生成该 response 时刻的 kv 的 revision.
- Raft_Term:  生成该 response 时刻，该 member 的 Raft term.

可以通过 Cluster_ID 或 Member_ID 字段以确保正在与预期的集群（成员）通信。  
可以通过 Revision 字段来了解 key-value store 的最新的 revision。当应用声明一个 historical revision 以进行 time travel query，并希望获取请求时的 latest revision 时尤其有用。  
可以使用 Raft_Term 来检测集群何时完成新的 leader election。

# Key-Value API
## System primitives

### Key-Value Pair
一个 KV pair 是 key-value API 可操作的最小单元。一个 Key-Value pair 的定义如下:
```go
message KeyValue {
    bytes key = 1;
    int64 create_revision = 2;
    int64 mod_revision = 3;
    int64 version = 4;
    bytes value = 5;
    int64 lease = 6;
}
```
- Version: 该 key 的 version。`delete` 会将 version 给 reset 为 0，任何修改都会增加该版本。
- Create_Revision: 对该 key 最后一次 create 的 revision。
- Mod_Revision: 对该 key 最后一次 modify 的  revision。
- Lease: 该 key 的租约 ID。没有租约时为 0。

Revision 使得可以按 creation 和 modification 时间对键进行排序，这对于分布式的 synchronization 很有用。etcd client 的 distributed shared locks 就是使用 creation revision 来等待锁所有权。类似地，modification revision 可以用于检测 software transactional memory read set conflicts 以及等待 leader election updates。

### Revisions
etcd 维护了一个 64-bit 的 cluster-wide 的计数器，它会在每次修改 key space 时递增，从而作为一个全局的逻辑时钟，实现对存储中的所有 updates 进行排序。

revision 对于 etcd3 的 MVCC 很有价值。etcd 对于历史记录的保留策略可以通过 admin configuration 进行细粒度的存储管理，通常 etcd3 会通过一个 timer 去丢弃了旧版本的 keys。通常 etcd3 群集会保留数小时的废弃的 keys。这为那些 long client disconnection 提供了可靠的处理：watchers 可以简单地从上一次观测到的 historical revision 中恢复。

### Key ranges
etcd3 的 data model 是将所有 keys 的索引作为一个 flat 的 binary key space，而不是将 keys 组织为目录形式的层次结构。因此 etcd3 也不是根据目录来 list keys，而是采用一个区间 [a, b)。

一个请求的 range 是由 `key` 和 `range_end` 两个字段来表示的。
- `key` 是该 range 的第一个 key，应该是非空的。
- `range_end` 是该 range 的最后一个 key 之后的键。
  - 如果 `range_end` 没有给定或为空，则该 range 被定义为只包含 `key`
  - 如果 `range_end` 是 `key` + 1，那么该 range 表示所有以 `key` 为前缀的键。
- 如果 `key` 和 `range_end` 都是 '/0'，那么该 range 表示所有键。
- 如果 `range_end` 是 '/0'，则该 range 代表所有大于或等于 `key` 的键。

## Range
etcd 中是通过 Range API 来获取 kv-store 中的 keys，它会有一个 RangeRequest:
```go
message RangeRequest {
    enum SortOrder {
        NONE    = 0; // default, no sorting
        ASCEND  = 1; // lowest target value first
        DESCEND = 2; // highest target value first
    }
    enum SortTarget {
        KEY = 0;
        VERSION = 1;
        CREATE = 2;
        MOD = 3;
        VALUE = 4;
    }

    bytes key = 1;
    bytes range_end = 2;
    int64 limit = 3;
    int64 revision = 4;
    SortOrder sort_order = 5;
    SortTarget sort_target = 6;
    bool serializable = 7;
    bool keys_only = 8;
    bool count_only = 9;
    int64 min_mod_revision = 10;
    int64 max_mod_revision = 11;
    int64 min_create_revision = 12;
    int64 max_create_revision = 13;
}
```
- Key, Range_End: The key range to fetch.
- Limit: 该请求返回的 keys 的最大数量。0 表示无限制。
- Revision: MVCC 请求的 point-in-time。该字段小于等于 0 时表示最新的值。如果请求的 Revision 已经被 compaction 删除则返回 ErrCompacted。
- Sort_Order: 排序方式
- Sort_Target: 排序所依据的字段
- Serializable: 
  - 默认情况下，Range 请求时 linearizable 的; 
  - 如果设置为 true，则以接受 stale read 为代价，在本地提供 serializable range request，而无需与群集中的其他节点达成共识。

Range 的调用会返回一个 RangeResponse
```go
message RangeResponse {
    ResponseHeader header = 1;
    repeated mvccpb.KeyValue kvs = 2;
    bool more = 3;
    int64 count = 4;
}
```
- Kvs: 符合 range request 的 kv-pairs，当使用 Count_Only 时 kvs 为空。
- More: 如果设置了 limit，则会通过这个字段表示是否还有更多匹配的 keys。


### Put
Keys 通过 Put 调用存储到 kv store 中。一个 PutRequest 如下:
```go
message PutRequest {
    bytes key = 1;
    bytes value = 2;
    int64 lease = 3;
    bool prev_kv = 4;
    bool ignore_value = 5;
    bool ignore_lease = 6;
}
```
- Lease: key 的 lease ID，为 0 表示没有租约。
- Prev_Kv: 设置了这个 flag 则会返回在这次 Put Request 之前的 kv pair。
- Ignore_Value: 设置该 flag 表示在不修改 value 的情况下更新 key。
- Ignore_Lease: 设置该 flag 表示在不修改 lease 的情况下更新 key。

```go
message PutResponse {
    ResponseHeader header = 1;
    mvccpb.KeyValue prev_kv = 2;
}
```

### Delete Range
```go
message DeleteRangeRequest {
    bytes key = 1;
    bytes range_end = 2;
    bool prev_kv = 3;
}
```
- Key, Range_End: The key range to delete.
- Prev_Kv: 设置了这个 flag 则会返回在这次 Delete Request 之前的 kv pair。

```go
message DeleteRangeResponse {
    ResponseHeader header = 1;
    int64 deleted = 2;
    repeated mvccpb.KeyValue prev_kvs = 3;
}
```
- Deleted: 被删除的 keys 数量。
- Prev_Kv: 本次 DeleteRange operation 所删除的 kv pairs。

## Transaction
一个事务可以在一个请求中原子地处理多个请求。这意味着 revision 在本次事务中只被递增一次，并且事务产生的所有事件将具有相同的 revision。然而，etcd3 禁止在一个事务中多次修改同一个键。

所有的事务都由一个 comparisions 的连接词来作保护，类似于一个 If statement。每个 comparison 会检查存储中的一个键，两个不同的 comparisions 可能是应用于相同或不同的键。所有的 comparisions 都是原子地 apply 的；如果所有的 comparisions 都是 true 则表示事务成功，etcd 会 apply 该事务的 `then / success` block; 否则表示失败，etcd 会 apply 事务的 `else / failure` block。

每个 comparison 被编码为一个 Compare 消息:
```go
message Compare {
    enum CompareResult {
        EQUAL = 0;
        GREATER = 1;
        LESS = 2;
        NOT_EQUAL = 3;
    }
    enum CompareTarget {
        VERSION = 0;
        CREATE = 1;
        MOD = 2;
        VALUE= 3;
    }
    CompareResult result = 1;
    // target is the key-value field to inspect for the comparison.
    CompareTarget target = 2;
    // key is the subject key for the comparison operation.
    bytes key = 3;
    oneof target_union {
        int64 version = 4;
        int64 create_revision = 5;
        int64 mod_revision = 6;
        bytes value = 7;
    }
}
```
- Result: 逻辑比较运算的类型
- Target: 需要比较的 key-value 的字段
- Key: 要比较的 key
- Target_Union: 要比较的 user-specified data

在处理完 comparison block 之后，事务会 apply 一个 request block。一个 block 是一个 RequestOp messages 的列表:
```go
message RequestOp {
    // request is a union of request types accepted by a transaction.
    oneof request {
        RangeRequest request_range = 1;
        PutRequest request_put = 2;
        DeleteRangeRequest request_delete_range = 3;
    }
}
```
- Request_Put: PutRequest, 其 keys 必须唯一，且不能与任何其他的 Puts 或 Deletes 存在相同的 key。
- Request_Delete_Range: DeleteRangeRequest, 不能与任何其他的 Puts 或 Deletes 存在相同的 key。

综上，一个事务通过一个 Txn API 的调用发起，这会发起一个 TxnRequest:
```go
message TxnRequest {
    repeated Compare compare = 1;
    repeated RequestOp success = 2;
    repeated RequestOp failure = 3;
}
```
- Compare: 一个 predicates 列表，标识了这个事务需要保证的连带条件。
- Success: 如果所有的 compare 结果为 true，则执行这些请求。
- Failure: 如果任何一个 compare 的结果为 false，则执行这些请求。

client 会从一个 Txn Call 中收到一个 TxnResponse message:
```go
message TxnResponse {
    ResponseHeader header = 1;
    bool succeeded = 2;
    repeated ResponseOp responses = 3;
}
```
Succeeded - 比较评估是否为真或假。
Responses - 响应列表，对应于应用Success块的结果（如果成功是真的）或Failure块的结果（如果成功是假的）。
Responses列表与应用RequestOp列表的结果相对应，每个响应被编码为ResponseOp。

- Succeeded: Compare 结果是否为 true。
- Responses: 根据 compare 的结果执行了一系列请求的 response
  - 每个 Response 被编码为一个 ResponseOp。

```go
message ResponseOp {
  oneof response {
    RangeResponse response_range = 1;
    PutResponse response_put = 2;
    DeleteRangeResponse response_delete_range = 3;
  }
}
```