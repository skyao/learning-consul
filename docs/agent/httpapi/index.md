Console Agent HTTP API
======================

> 注：内容翻译自官网资料 [HTTP API](https://www.consul.io/docs/agent/http.html)

cosul的主要接口是RESTful HTTP API.这个API可以用于在节点，服务，检查，配置等上执行CRUD操作。端点(endpoint)是带版本的,以便可以在不打破向后兼容的情况下修改。

每个端点管理consul不同的一个方面：

- [acl]() - Access Control Lists / 访问控制列表
- [agent]() - Consul Agent / consul中介
- [catalog]() - Nodes and services / 节点和服务
- [coordinate]() - Network coordinates / 网络协同
- [event]() - User Events / 用户事件
- [health]() - Health checks / 健康检查
- [kv]() - Key/Value store / 键值存储
- [query]() - Prepared Queries / 已准备的查询
- [session]() - Sessions / 会话
- [status]() - Consul system status / consul系统状态

这些中的每一个在上面的链接中都有详细的文档描述。consul也有一定数量的内部API故意没有给文档以便修改。

# 阻塞查询

某些端点支持被称为"阻塞查询"的特性。阻塞查询被用于等待可能的变化，使用长轮询(long polling)。

不是所有的端点都支持阻塞，但是这些在文档中都被清晰指明。任何支持阻塞的端点将设置HTTP 头 X-Consul-Index，一个唯一标记，表示当前被查询资源的状态。在随后的对这个资源的请求中，客户端可以设置index查询字符串参数到X-Consul-Index的值中，指示客户端希望等待这个index之后的任何变化子序列。

除index之外，支持阻塞的端点也将需要一个等待参数来指定这个阻塞请求的最大期限。最大限制为10分钟。如果没有设置，等待时间默认为5分钟。这个值可以用"10s"或"5m"(例如，分别是10秒或者5分钟)的形式来指定。

关键提示：阻塞式请求的返回并不保证一定有变化。可能是到达超时时间或者是有一个不影响查询结果的幂等写。

# 一致性模型

大部分读查询端点支持多个级别的一致性。因为没有策略可以适合所有客户端需要，这些一致性模型容许用户在如何平衡分布式系统内在的权衡上有最大的发言权。

这三个读模型是：

- default / 默认 - 如果没有指定，几乎所有情况下默认是强一致性。但是，有一个小窗口，在此期间新leader可能被选举出来而老的leader可能提供过时的值。权衡在于读很快但是有值过时的潜在风险。导致过时读取的条件很难触发，大部分客户端应该不需要担心这种情况。另外，注意这个竞态条件(race condition)仅仅发生在读，而不是写。

- consistent / 一致性 - 这个模式是不带警告的强一致性。它需要leader和法定人数的伙伴核实它依然是leader。这会引入额外的到所有服务器节点的往返流程(round-trip)。权衡在于因为额外往返流程增加的延迟。大部分客户端不应该使用它除非他们不能容忍过时读取。

- stale / 过时 - 这个模型容许任何服务器提供读取服务，无论它是不是leader。这意味着读可能任意的过时；不过，结果通常和leader保持50毫秒的一致性。权衡在于非常快并且读可扩展，但是有更高的过时值的可能性。因为这个模式容许在没有leader的情况下读取，不可到达的集群也将可以响应查询。

为了转换这些模式，可以在请求上提供stale或者consistent查询参数。同时提供两个则会报错。

为了支持限制数据的可以接受的过时,response提供X-Consul-LastContact头,包含单位为毫秒的时间,表示服务器和leader节点联系的最后时间。X-Consul-KnownLeader 头也指明是否有已知的leader。这些可以被客户端用来评估结果的过时性并采取适当的行动。

# 格式化的JSON输出

默认，所有HTTP API请求的输出最小化的JSON。如果客户端在查询字符串上传递pretty，将返回格式化的JSON。

## ACL

在consul中一些端点使用或者要求访问控制列表记号(ACL tokens)来操作。agent通过使用acl_token配置选项可以配置为在请求中使用默认记号。然而，这个记号也可以通过使用 X-Consul-Token 请求头或者token查询参数来为每个请求单独指定。请求头优先于默认记号，而查询字符串参数优先于所有。

