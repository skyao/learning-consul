Consul架构
===========

> 内容翻译自 [CONSUL ARCHITECTURE](https://www.consul.io/docs/internals/architecture.html)

Consul是一个复杂的系统，拥有多个不同的演进中的部分。为了帮助Consul的用户和开发人员形成consul如何工作的内在模型，这个页面讲述了系统架构。

> 高级主题！这个页面覆盖consul内部的技术细节。不需要了解这些细节来有效操作和使用consul。在这里描述这些细节是为了那些希望了解这些的人，让他们无需深入源代码。

# 术语

在阐述架构前，提供术语表以帮助澄清将要阐述的内容：

- Agent / 代理

	agent是在consul集群的每个成员上长时间运行的后台进程。通过执行"consul agent"启动。agent可以运行在客户端或者服务端模式。由于所有节点必须运行agent，和节点关联就更容易了，不管节点是客户端还是服务器，不过还有agent的其他实例。所有agent可以运行 DNS 或者 HTTP 接口，并负责运行检查和保持服务同步。

- Client / 客户端

	client是转发所有RPC请求到服务器的agent。client相对而言是无状态的。client进行的唯一活动是参与LAN gossip 池。这只需要极轻微的资源并只消耗少量的网络带宽。

- Server / 服务器

	server是一个职责扩展的agent，包括参与Raft团队，维持集群状态，响应RPC查询，和其他数据中心交互WAN gossip和转发请求到leader或者远程数据中心。

- Datacenter / 数据中心

	虽然数据中心的定义看上去很明显，依然还是有些细节必须考虑。例如，在EC2中，多个可到达的zone是否考虑组成一个单一的数据中心？我们定义数据中心为这样的网络环境：私有，低延迟，高带宽。这排除了跨越公共英特网的通讯，但是，在我们看来，在单个EC2 区域中的多个可到达zone可以考虑为单个数据中心的一部分。

- Consensus / 一致（共识）

	在我们的文档中，使用 consensus 来表示不仅有对被选举的leader的认可，而且有对事务顺序的认可。由于这些事务被应用到 [有限状态机](https://en.wikipedia.org/wiki/Finite-state_machine)，我们的consensus定义暗示被复制状态机的一致性。在 [Wikipedia](https://en.wikipedia.org/wiki/Consensus) 上有Consensus的更多详细内容，而我们的实现在 [这里](https://www.consul.io/docs/internals/consensus.html) 描述。

- Gossip

	Consul构建在Serf之上，Serf提供完全的用于多个用途的 gossip 协议. Serf 提供成员关系，失败检测，还有事件广播。我们对这些的使用在 [gossip文档](https://www.consul.io/docs/internals/gossip.html) 中有更多描述。这足以了解到gossip包含了随机节点到节点(node-to-node)通讯, 首选UDP。

- LAN Gossip / 局域网 Gossip

	和局域网gossip池相关，包含在同一个局域网或者数据中心中的所有
	Refers to the LAN gossip pool which contains nodes that are all located on the same local area network or datacenter.

WAN Gossip - Refers to the WAN gossip pool which contains only servers. These servers are primarily located in different datacenters and typically communicate over the internet or wide area network.

RPC - Remote Procedure Call. This is a request / response mechanism allowing a client to make a request of a server.

10,000 foot view

From a 10,000 foot altitude the architecture of Consul looks like this:

Consul Architecture

Let's break down this image and describe each piece. First of all, we can see that there are two datacenters, labeled "one" and "two". Consul has first class support for multiple datacenters and expects this to be the common case.

Within each datacenter, we have a mixture of clients and servers. It is expected that there be between three to five servers. This strikes a balance between availability in the case of failure and performance, as consensus gets progressively slower as more machines are added. However, there is no limit to the number of clients, and they can easily scale into the thousands or tens of thousands.

All the nodes that are in a datacenter participate in a gossip protocol. This means there is a gossip pool that contains all the nodes for a given datacenter. This serves a few purposes: first, there is no need to configure clients with the addresses of servers; discovery is done automatically. Second, the work of detecting node failures is not placed on the servers but is distributed. This makes failure detection much more scalable than naive heartbeating schemes. Thirdly, it is used as a messaging layer to notify when important events such as leader election take place.

The servers in each datacenter are all part of a single Raft peer set. This means that they work together to elect a single leader, a selected server which has extra duties. The leader is responsible for processing all queries and transactions. Transactions must also be replicated to all peers as part of the consensus protocol. Because of this requirement, when a non-leader server receives an RPC request, it forwards it to the cluster leader.

The server nodes also operate as part of a WAN gossip pool. This pool is different from the LAN pool as it is optimized for the higher latency of the internet and is expected to contain only other Consul server nodes. The purpose of this pool is to allow datacenters to discover each other in a low-touch manner. Bringing a new datacenter online is as easy as joining the existing WAN gossip. Because the servers are all operating in this pool, it also enables cross-datacenter requests. When a server receives a request for a different datacenter, it forwards it to a random server in the correct datacenter. That server may then forward to the local leader.

This results in a very low coupling between datacenters, but because of failure detection, connection caching and multiplexing, cross-datacenter requests are relatively fast and reliable.

Getting in depth

At this point we've covered the high level architecture of Consul, but there are many more details for each of the subsystems. The consensus protocol is documented in detail as is the gossip protocol. The documentation for the security model and protocols used are also available.

For other details, either consult the code, ask in IRC, or reach out to the mailing list.