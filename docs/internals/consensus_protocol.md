一致性协议
========

> 注：内容翻译自官网文档 [CONSENSUS PROTOCOL](https://www.consul.io/docs/internals/consensus.html)

Consul使用 [一致性协议(consensus protocol)](https://en.wikipedia.org/wiki/Consensus_(computer_science)) 来提供一致性(如在CAP中定义的)。一致性协议基于"[Raft: In search of an Understandable Consensus Algorithm](https://ramcloud.stanford.edu/wiki/download/attachments/11370504/raft.pdf)".对于Raft形象的解释，请看[The Secret Lives of Data](The Secret Lives of Data).

	特别备注：上面这个 The Secret Lives of Data 的内容和展现形式都非常的精彩，强烈推荐阅读，绝不要错过！

> 高级主题！这个页面覆盖consul内部的技术细节。有效操作和使用consul并不需要了解这些细节。在这里描述这些细节是为了那些希望了解这些的人，让他们无需深入源代码。

# Raft协议概述

Raft是一个基于Paxos的一致性算法。相比Paxos，Raft被设计为有更少的状态和更简单更容易理解的算法。

这里有一些在讨论Raft时需要了解的关键术语：

- Log -  在Raft系统中最主要的工作单元是日志记录(log entry). 一致性的问题可以分解到复制的日志。日志是排序后的记录队列。如果所有成员认可记录和他们的顺序，我们认为日志是一致的。

- FSM - 有限状态机。FSM 是有限状态的集合，这些状态之间相互转换。和新日志被应用一样，FSM被容许在状态间转换。拥有相同日志序列的应用必须归结于相同的状态，意味着行为必须是确定性的。

- Peer set / 端集合 - peer set 是参与日志复制的所有成员的集合。对于consul而言，所有服务器节点都在本地数据中心的peer set中。

- Quorum / 法定人数 - 法定人数是指peer set中的大多数： 对于大小为n的集合，法定人数需要至少(n/2)+1个成员。例如，如果在peer set中有5个成员，我们将需要3个节点来达到法定人数。如果无法达到节点的法定人数，无论任何原因，集群变得不可到达从而不能提交新日志。

- 提交的记录 - 当记录被持久性存储在法定人数的节点上时，记录被认为是提交的。一旦记录被提交，记录就可以被应用。

- Leader - 在任何时刻， peer set选举一个单一节点作为leader。leader负责接待新的日志记录，复制到跟随者，并且在记录被认为已提交时进行管理。

Raft是一个复杂协议，不在这里详细覆盖(对于那些需要更加综合对待的同学，完全的规范可以看这个[论文](https://ramcloud.stanford.edu/wiki/download/attachments/11370504/raft.pdf)). 无论如何，我们将试图提供一个高级别的描述，对于构建一个原始模型可能有用。

Raft节点通常是三种状态之一：follower/跟随者, candidate/候选者, 或 leader。所有的节点最初作为跟随者启动。在这个状态下，节点可以从leader接收日志记录并投票。如果有段时间没有收到记录，节点自我提升为候选者状态。在候选者状态，节点从他们的端(peer)要求投票。如果候选者收到法定人数的投票，那么它被提升为leader。leader必须接受新的日志记录并且复制到所有其他跟随者。另外，如果脏读不可接受，所有请求必须在leader上被执行。

一旦集群拥有leader，它可以接受新的日志记录。客户端可以请求leader附加一个新的日志记录（在Raft看来，日子记录是一个不透明的二进制blob。leader随后写记录到持久存储并尝试复制到法定人数的跟随者。一旦日志记录被认为被提交，它可以被应用到一个有限状态机。有限状态机随应用而不同。在Consul的案例中，我们使用BoltDB来维护集群状态。

明显，没有必要容许复制日志无限增长。raft提供一个机制，快速当前状态并压缩日志。因为FSM抽象，存储FSM的状态必然导致同样的状态作为就日志的重放。这容许Raft捕获FSM状态

Obviously, it would be undesirable to allow a replicated log to grow in an unbounded fashion. Raft provides a mechanism by which the current state is snapshotted and the log is compacted. Because of the FSM abstraction, restoring the state of the FSM must result in the same state as a replay of old logs. This allows Raft to capture the FSM state at a point in time and then remove all the logs that were used to reach that state. This is performed automatically without user intervention and prevents unbounded disk usage while also minimizing time spent replaying logs. One of the advantages of using BoltDB is that it allows Consul to continue accepting new transactions even while old state is being snapshotted, preventing any availability issues.

Consensus is fault-tolerant up to the point where quorum is available. If a quorum of nodes is unavailable, it is impossible to process log entries or reason about peer membership. For example, suppose there are only 2 peers: A and B. The quorum size is also 2, meaning both nodes must agree to commit a log entry. If either A or B fails, it is now impossible to reach quorum. This means the cluster is unable to add or remove a node or to commit any additional log entries. This results in unavailability. At this point, manual intervention would be required to remove either A or B and to restart the remaining node in bootstrap mode.

A Raft cluster of 3 nodes can tolerate a single node failure while a cluster of 5 can tolerate 2 node failures. The recommended configuration is to either run 3 or 5 Consul servers per datacenter. This maximizes availability without greatly sacrificing performance. The deployment table below summarizes the potential cluster size options and the fault tolerance of each.

In terms of performance, Raft is comparable to Paxos. Assuming stable leadership, committing a log entry requires a single round trip to half of the cluster. Thus, performance is bound by disk I/O and network latency. Although Consul is not designed to be a high-throughput write system, it should handle on the order of hundreds to thousands of transactions per second depending on network and hardware configuration.

# Raft in Consul

Only Consul server nodes participate in Raft and are part of the peer set. All client nodes forward requests to servers. Part of the reason for this design is that, as more members are added to the peer set, the size of the quorum also increases. This introduces performance problems as you may be waiting for hundreds of machines to agree on an entry instead of a handful.

When getting started, a single Consul server is put into "bootstrap" mode. This mode allows it to self-elect as a leader. Once a leader is elected, other servers can be added to the peer set in a way that preserves consistency and safety. Eventually, once the first few servers are added, bootstrap mode can be disabled. See this guide for more details.

Since all servers participate as part of the peer set, they all know the current leader. When an RPC request arrives at a non-leader server, the request is forwarded to the leader. If the RPC is a query type, meaning it is read-only, the leader generates the result based on the current state of the FSM. If the RPC is a transaction type, meaning it modifies state, the leader generates a new log entry and applies it using Raft. Once the log entry is committed and applied to the FSM, the transaction is complete.

Because of the nature of Raft's replication, performance is sensitive to network latency. For this reason, each datacenter elects an independent leader and maintains a disjoint peer set. Data is partitioned by datacenter, so each leader is responsible only for data in their datacenter. When a request is received for a remote datacenter, the request is forwarded to the correct leader. This design allows for lower latency transactions and higher availability without sacrificing consistency.

# Consistency Modes

Although all writes to the replicated log go through Raft, reads are more flexible. To support various trade-offs that developers may want, Consul supports 3 different consistency modes for reads.

The three read modes are:

default - Raft makes use of leader leasing, providing a time window in which the leader assumes its role is stable. However, if a leader is partitioned from the remaining peers, a new leader may be elected while the old leader is holding the lease. This means there are 2 leader nodes. There is no risk of a split-brain since the old leader will be unable to commit new logs. However, if the old leader services any reads, the values are potentially stale. The default consistency mode relies only on leader leasing, exposing clients to potentially stale values. We make this trade-off because reads are fast, usually strongly consistent, and only stale in a hard-to-trigger situation. The time window of stale reads is also bounded since the leader will step down due to the partition.

consistent - This mode is strongly consistent without caveats. It requires that a leader verify with a quorum of peers that it is still leader. This introduces an additional round-trip to all server nodes. The trade-off is always consistent reads but increased latency due to the extra round trip.

stale - This mode allows any server to service the read regardless of if it is the leader. This means reads can be arbitrarily stale but are generally within 50 milliseconds of the leader. The trade-off is very fast and scalable reads but with stale values. This mode allows reads without a leader meaning a cluster that is unavailable will still be able to respond.

For more documentation about using these various modes, see the HTTP API.
