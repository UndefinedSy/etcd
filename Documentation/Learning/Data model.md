etcd 旨在高可用强一致地存储不频繁更新的数据，提供可靠的 watch queries。其提供了对于 KV 的 MVCC，支持查看历史事件（time travel queires）

etcd 不会原地更新数据，而是会在更新中生成一个新的 updated structre。因此所有的旧版本仍然可访问且可 watch。

## Logical View
etcd 存储的逻辑视图是一个 flat 的 binary key space，key space 有一个对于 byte string keys 的 lexically sorted index，以加速 range queries。

初始 revision 是 1，每个 atomic mutative operation（一个事务中可以包含若干 operations）会在这个 key space 上创建一个新的 revision。revisions 本身也是有索引的，所以可以高效的 ranging over revisions with watchers。

对一个 key 的删除操作会生成一个 key tombstone，将该 key 的 version 给 reset 为 0 来结束 key 的当前 generation。一旦发生 compaction，任何在 compaction revision 前结束的 generation 都将被删除，而在 compaction revision 前 set 的值，除了 latest one 以外，都将被删除。

## Physical View
etcd 将数据存储为一个持久化的 b+tree 中的 kv-pair。存储状态的每个 revision 只包含当前 revision 与前一个 revision 的 $\delta$，以提高效率。一个 revision 可能对应于 b+tree 中的多个 keys。

一个 kv-pair 的 key 是一个 3 元组 `(major, sub, type)`。
- major: 该键的 store revision
- sub: 用于区分同一 revision 中的 keys。
- type: 可选的后缀，用于标识一些特殊值（例如，如果一个值包含有 tombstone，则这里为 **t**）。

一个 kv-pair 的 value 包含了从上一个 revision 到现在的修改 $\delta$。因为 b+tree 本身是按 key 的词法字节序排序的，因此对 revision $\delta$s 的 range query 是很快的。

etcd 还在内存中有一个 secondary btree index，以加速对 key 的 range queries。在这个 btree index 中:
- key 是暴露给用户的 key
- value 是一个指向 persistent b+tree 的指针。
  - compaction 会删除掉 dead pointers。