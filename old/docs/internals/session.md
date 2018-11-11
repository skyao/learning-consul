会话
=====

> 注： 内容翻译自官网文档 [sessions](https://www.consul.io/docs/internals/sessions.html)

Consul提供一个会话机制，可以用来构建分布式锁。会话被设计用来提供颗粒状锁定并很大程度上受到 [The Chubby Lock Service for Loosely-Coupled Distributed Systems](http://research.google.com/archive/chubby.html) 的启发.

> 高级主题！这个页面覆盖consul内部的技术细节。有效操作和使用consul并不需要了解这些细节。在这里描述这些细节是为了那些希望了解这些的人，让他们无需深入源代码。

# 会话设计

Consul中的会话提现为有非常明确语义的契约。当会话被构建时，可能会提供一个节点名，一个健康检查列表，一个行为，一个TTL，和一个lock-delay。一个新构建的会话伴随有一个命名ID，可以用来标记它。这个ID可以和键值对存储一起使用来获得锁：用于乎斥的顾问机制。

下面的图展示这些组件之间的关系：

![](https://www.consul.io/assets/images/consul-sessions-56bc7006.png)

consul提供的契约是在以下任何情况，会话将失效：

- 节点被注销
- 任何健康检查被注销
- 任何健康检查转为严重状态
- 会话被显式销毁
- TTL(Time To Live/存活时间)过期，如果适用

当会话失效时，会话被销毁并不能再被使用。对关联的锁发生什么取决于创建时刻指定的行为。consul支持release/发布 and delete/删除行为。如果没有指定模式为发布行为。

如果发布行为被使用，任意锁

If the release behavior is being used, any of the locks held in association with the session are released, and the ModifyIndex of the key is incremented. Alternatively, if the delete behavior is used, the key corresponding to any of the held locks is simply deleted. This can be used to create ephemeral entries that are automatically deleted by Consul.

While this is a simple design, it enables a multitude of usage patterns. By default, the gossip based failure detector is used as the associated health check. This failure detector allows Consul to detect when a node that is holding a lock has failed and to automatically release the lock. This ability provides liveness to Consul locks; that is, under failure the system can continue to make progress. However, because there is no perfect failure detector, it's possible to have a false positive (failure detected) which causes the lock to be released even though the lock owner is still alive. This means we are sacrificing some safety.

Conversely, it is possible to create a session with no associated health checks. This removes the possibility of a false positive and trades liveness for safety. You can be absolutely certain Consul will not release the lock even if the existing owner has failed. Since Consul APIs allow a session to be force destroyed, this allows systems to be built that require an operator to intervene in the case of a failure while precluding the possibility of a split-brain.










