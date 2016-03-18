Consul Agent/Consul代理
===========

> 注：内容翻译自 官网文档 [CONSUL AGENT](https://www.consul.io/docs/agent/basics.html)

Consul agent 是consul的核心进程。agent维护成员信息，注册服务，运行检查，响应请求等等。agent必须运行在每个作为consul集群一部分的节点上。

任何agent都将运行于两个模式之一：client 或者 server。server节点作为一致性法定人数的一部分担当额外职责。这些节点参与Raft并提供强一致性和失败情况下的可用性。服务器节点上的更高负载意味着通常他们应该运行在专用的实例上--他们比client节点更加资源敏感。client节点组成集群的大部分，并且他们是非常轻量级，因为他们连接server节点来处理大部分请求并且维护他们自身的非常少的状态。

# 运行Agent

agent是用consul agent命令启动。这个命令会阻塞，一直运行或者知道被告之推出。agent命令接受多个配置选项，但是大部分有默认值。

当运行consul agent时，你将看到类似这样的输出：

```bash
$ consul agent -data-dir=/tmp/consul
==> Starting Consul agent...
==> Starting Consul agent RPC...
==> Consul agent running!
       Node name: 'Armons-MacBook-Air'
      Datacenter: 'dc1'
          Server: false (bootstrap: false)
     Client Addr: 127.0.0.1 (HTTP: 8500, DNS: 8600, RPC: 8400)
    Cluster Addr: 192.168.1.43 (LAN: 8301, WAN: 8302)
           Atlas: (Infrastructure: 'hashicorp/test' Join: true)

==> Log data will now stream in as it occurs:

    [INFO] serf: EventMemberJoin: Armons-MacBook-Air.local 192.168.1.43
...
```

consul agent输出一些重要的信息：

- Node name: 节点名称，这是agent的唯一名称。默认是机器的hostname，但是可以使用 -node 标记来定制它。

- Datacenter: 数据中心，这是agent被配置运行所在的数据中心。consul对多数据中心有第一流的支持。然后，为了有效工作，每个节点必须设置为报告他的数据中心。-dc 标记可以用来设置数据中心。对于单数据中心配置，agent将默认为"dc1".

- Server: 这表明agent是以server还是client节点运行。server节点有额外的担当如参与一致性法定人数，存储集群状态，并处理请求。另外，服务器可能处于"bootstrap"模式。不能有多个server处于bootstrap模式因为这样会将集群置于不一致的状态。

- Client Addr: 用于client连接到agent的地址。包括HTTP，DNS和RPC接口的端口。RPC地址用于其他consul命令(例如consul members，consul join等)，查询并控制运行中的agent。默认，仅仅绑定到localhost。如果修改这个地址或者端口，将不得不在运行如consul members命令的时候指定-rpc-addr来指示如何达到agent。其他应用也可以使用RPC地址和端口来控制consul。

- Cluster Addr: 用于集群中的consul agent之间通讯的地址和端口的集合。并非集群中的所有consul agent都必须使用同样的端口，但是这个地址必须让所有其他节点可以到达。

- Atlas: 显示节点注册时带上的 [Atlas infrastructure/Atlas基础设置](https://atlas.hashicorp.com/)。也会标明是否开启了auto-join。Atlas基础设置通过使用-atlas来设置，而auto-join通过设置-atlas-join来开启。

# 停止agent

agent可以用两种方式停止：优雅或者强制。为了优雅的挂起agent，给进程发送一个终端信号(通常在终端中Ctrl-C或者运行kill -INT consul_pid)。在优雅退出时，agent首先告之集群它试图离开集群。这样，其他集群成员通知集群节点已经离开。

或者，可以通过发送kill 信号强行kill agent。当强制kill时，agnet立即结束。集群的剩余部分将最终(通常在几秒内)检测到节点已死并通知集群节点已经失败。

特别重要：server节点被容许优雅退出以便在可用性上影响最小，因为服务器舍弃(leaves)了一致性法定人数。(注：我的理解是，优雅退出时，一致性法定人数会相应减少，因此后续投票时法定人数和server节点的数量是准确对应的。而强制退出时，法定人数不变而实际这个server节点已经退出，因此后续的投票始终会缺这个server node的票)。

对于client agent，节点失败和节点离开的差别对于使用场景可能并不重要。例如，对于web 服务器和负载均衡搭建，都将导致相同的后果：web节点被从负载均衡池中移除。

# 生命周期

consul集群中的每个agent都通过一个生命周期。理解这个生命周期有助于构建agent和集群的交互和集群如何处理节点的内在模型。

当agent初始启动时，它不知道集群中的任何其他节点。为了发现它的伙伴，他必须加入集群。这是通过join命令或者在启动时通过提供适当的配置做auto-joi来完成的。一旦节点加入，这个信息被传播到整个集群，意味着所有节点将最终知晓彼此。如果agent是服务器，已存在的服务器将开始复制给新的节点。

就网络失败而言，一些节点可能对于其他节点无法到达。在这种情况下，无法到达的节点被标记为失败。无法区分网络失败和agent崩溃，因此两种情况被同样处理。一旦节点被标记为失败，这个信息被更新在服务目录(catalog)。注意：有一些细微差别，因为这个更新仅当服务器还能构成法定人数时才有可能。一旦网络恢复或者崩溃的agent重启，集群将自我修复并不再标记节点为失败。catalog中的健康检查也将更新来反应这个。

当节点离开时，它指出它打算这样做，而集群标记这个节点将要离开。与失败场景不同，节点提供的所有服务立即注销。如果agent是server，到它的复制将被停止。

为了阻止死亡节点的积累(节点处于失败或者离开状态)，consul将死亡节点自动移除到catalog外。这个过程称为收割(reaping)。这目前是通过不可配置的72小时时间间隔来完成。Reaping类似于离开，导致所有相关服务被注销。
