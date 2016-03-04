Consul
===========

# 介绍

Consul是HashiCorp公司推出的开源工具，在 [Consul的官网](https://www.consul.io/), consul是如此介绍自己的：

> Service discovery and configuration made easy / 简化服务发现和配置
> Distributed, highly available, and datacenter-aware / 分布式，高可用，多数据中心

## 主要特性

> 以下内容翻译自官网 [首页](https://www.consul.io/) 和介绍资料 [consul介绍/INTRODUCTION TO CONSUL](https://www.consul.io/intro/index.html)。

- 服务发现

	![](https://www.consul.io/assets/images/feature-discovery@2x-e2a08445.png)

    Consul简化简化了服务注册和发现，可以使用DNS或者HTTP接口。也可以注册外部服务比如SaaS提供商。

	Consul的客户端可以提供服务，例如API或者mysq，而其他客户端可以使用consul来发现给定服务的提供者。通过使用DNS或者HTTP，应用可以轻易的找到他们依赖的服务。

- 失败检测

	![](https://www.consul.io/assets/images/feature-health@2x-50ca2d94.png)

	和服务发现配套的健康检查阻止将请求路由到不健康的主机并使得服务易于提供熔断器。

	Consul客户端可以提供任意数量的监控检查，可以和给定服务关联(webserver是否返回200OK)，或者和本地节点关联(内存使用率是否低于90%)。这个信息可以被操作者使用来监控集群健康，也用于服务发现组件路由请求远离不健康站点。

- 多数据中心

	![](https://www.consul.io/assets/images/feature-multi@2x-9fea2a34.png)

	Consul 很容易扩展到多个数据中心，不需要复杂的配置。可以在其他数据中心查找服务，或者保持请求本地。

	支持多数据中心意味着consul的用户不必担心构建额外的抽象层来扩展到多个地域。

- 键值对存储

	![](https://www.consul.io/assets/images/feature-config@2x-d09628e4.png)

	灵活的键值对存储，用于动态配置，特性标记，协调，leader选举等. 支持Long poll用于配置变更的准实时通知。

	应用可以将consul的分层键值对存储用于任何目的。简单的HTTP API易于使用。

consul被设计为对DevOps社区和应用开发者友好，使得它可以完美的用于现代而灵活的基础设施.

# Consul的主要架构

Consul是分布式的高可用系统。

每个提供服务给consul的节点都运行有一个consul的agent。运行agent对于发现其他服务或者获取/设置键值对数据并非必须。agent负责做节点上的服务的健康检查也包括节点自己。

agent和一个或者多个consul服务器对话。数据存储并同步在consul服务器上。服务器自行选择leader。虽然consul可以单机工作，推荐3到5台consul以避免失败场景导致数据丢失。推荐每个数据中心一个consul服务器集群。

需要发现其他服务或者节点的基础设施的组件可以查询任何consul 服务器或者consul agent。agent自动将请求转发给服务器。

每个数据中心运行consul服务器的一个集群。当跨数据中心的服务发现或者配置请求发生时，本地consul服务器转发请求到远程数据中心并返回结果。


