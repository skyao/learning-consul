Consul架构
===========

> 注：内容翻译自 [CONSUL ARCHITECTURE](https://www.consul.io/docs/internals/architecture.html)

Consul是一个复杂的系统，拥有多个不同的演进中的部分。为了帮助Consul的用户和开发人员形成consul如何工作的内在模型，这个页面讲述了系统架构。

> 高级主题！这个页面覆盖consul内部的技术细节。有效操作和使用consul并不需要了解这些细节。在这里描述这些细节是为了那些希望了解这些的人，让他们无需深入源代码。

# 术语

在阐述架构前，提供术语表以帮助澄清将要阐述的内容：

- Agent / 代理

	agent是在consul集群中的每个成员上长时间运行的后台进程。通过执行"consul agent"启动。agent可以运行在客户端或者服务端模式。由于所有节点必须运行agent，和节点关联就更容易了，不管节点是客户端还是服务器(不过还有agent的其他实例)。所有agent可以运行 DNS 或者 HTTP 接口，并负责运行检查和保持服务同步。

- Client / 客户端

	client是转发所有RPC请求到服务器的agent。client相对而言是无状态的。client进行的唯一活动是参与LAN gossip 池。这只需要极轻微的资源并只消耗少量的网络带宽。

- Server / 服务器

	server是一个职责扩展的agent，包括参与Raft团队，维持集群状态，响应RPC查询，和其他数据中心交互WAN gossip和转发请求到leader或者远程数据中心。

- Datacenter / 数据中心

	虽然数据中心的定义看上去很明显，依然还是有些细节必须考虑。例如，在EC2中，多个可到达的zone是否考虑组成一个单一的数据中心？我们定义数据中心为这样的网络环境：**私有，低延迟，高带宽**。这排除了跨越公共英特网的通讯，但是，在我们看来，在单个EC2 区域中的多个可到达zone可以考虑为单个数据中心的一部分。

- Consensus / 一致性

	在我们的文档中，使用 consensus /一致性 来表示不仅有对被选举的leader的认可，而且有对事务顺序的认可。由于这些事务被应用到 [有限状态机](https://en.wikipedia.org/wiki/Finite-state_machine)，我们的consensus定义暗示被复制状态机的一致性。在 [Wikipedia](https://en.wikipedia.org/wiki/Consensus) 上有Consensus的更多详细内容，而我们的实现在 [这里](https://www.consul.io/docs/internals/consensus.html) 描述。

- Gossip

	Consul构建在Serf之上，Serf提供完整的用于多个用途的 gossip 协议. Serf 提供成员关系，失败检测，还有事件广播。我们对这些的使用在 [gossip文档](https://www.consul.io/docs/internals/gossip.html) 中有更多描述。这足以了解到gossip包含了随机节点到节点(node-to-node)通讯, 首选UDP。

- LAN Gossip / 局域网 Gossip

	和局域网gossip池相关，包含在同一个局域网或者数据中心中的所有节点。

- WAN Gossip / 广域网 Gossip

	和广域网gossip池相关，仅仅包含服务器。这些服务器部署在不同的数据中心并且通常在英特网或者广域网上通讯。

- RPC / 远程程序调用

	远程程序调用(Remote Procedure Call). 这是一个请求/应答机制，容许客户端产生服务器请求.

# 高空鸟瞰

从一万英尺的高度看，consul的架构类似这样：

![consul的架构](https://www.consul.io/assets/images/consul-arch-5d4e3623.png)

让我们分解这个图片并描述每一块。首先，我们可以看到有两个数据中心，标签为"Datacenter 1"和"Datacenter 2"。consul对多数据中心提供最高等级的支持并预期这将成为普遍情况。

在每个数据中心内，混合有客户端和服务器。预期由3到5个服务器。强调在失败情况下的可达到和性能之间的平衡，因为添加更多机器会导致一致性变得日益缓慢。不过，对于客户端的数量没有限制，而且可以很容易的扩展到成千上万。

在一个数据中心内的所有节点加入一个gossip 协议。这意味着对于一个给定的数据中心，有一个包含所有节点的gossip池。这服务于几个目的：

1. 首先，不需要用服务器地址来配置客户端；发现是自动完成的。
2. 其次，检测节点失败的工作不是放置在服务器上，而是分布式的。这使得失败检测比幼稚的心跳机制更加可扩展。
3. 另外，当重要事件比如leader选举发生时它被用来作为消息层(messaging layer)来通知。

每个数据中心中的服务器是单个Raft 端集合(peer set)的所有组成部分。这意味着他们一起工作来选举leader，一个有额外职责的被选定的服务器。leader负责处理所有请求和事务。事务也必须复制到所有作为一致协议一部分的端(peer)。因为这个要求，当一个非leader服务器收到RPC请求时，它转发请求到集群leader。

服务器节点也作为广域网 gossip 协议池的一部分运作。这个池和局域网池有所不同，因为它为因特网的高延迟做了优化并预期只包含其他consul服务器(注：特指在其他数据中心中的consul服务器，见图中的"Datacenter 2")节点。这个池的目的是容许数据中心们以低接触(low-touch)风格相互发现。上线一个新的数据中心和加入现有的广域网gossip一样容易。因为服务器都在全体经营这个池，池可以开启跨数据中心请求。当服务器收到一个给不同数据中心的请求时，它转发请求到正确的数据中心中的任意服务器。这个服务器可能随即转发给本地leader。

这使得数据中心之间的耦合的非常低。而因为失败检测，连接缓存和多路通讯，跨数据中心请求相对比较快而可靠。

# 深入

此刻我们已经覆盖了consul的高层架构，但是对于每个子系统还有很多的细节。[一致性协议(consensus protocol)](https://www.consul.io/docs/internals/consensus.html) 被作为 [gossip协议](https://www.consul.io/docs/internals/gossip.html) 给做了详细的文档。安全模式的 [文档](https://www.consul.io/docs/internals/security.html) 和使用的协议也同样可以得到。

对于其他细节，可以查阅代码，在IRC提问，或者求助于邮件列表。
