Agent HTTP端点
=============

> 注： 内容翻译自官网文档 [AGENT HTTP ENDPOINT](https://www.consul.io/docs/agent/http/agent.html)

Agent端点用于和本地consul agent交互。通常，服务和检查被注册到agent，而agent则担负起和集群保持数据同步的职责。例如，agent用catalog注册服务和检查，并执行[反熵/anti-entropy](../../docs/internals/anti-entropy.html)从断供(outages)中恢复。

支持下列端点：

* [`/v1/agent/checks`](#agent_checks) : 返回本地agent正在管理的检查
* [`/v1/agent/services`](#agent_services) : 返回本地agent正在管理的服务
* [`/v1/agent/members`](#agent_members) : 返回被本地serf agent看到的成员(注：这里的serf做何解？)
* [`/v1/agent/self`](#agent_self) : 返回本地节点配置
* [`/v1/agent/maintenance`](#agent_maintenance) : 管理节点维护方式
* [`/v1/agent/join/<address>`](#agent_join) : 触发本地agent添加一个节点
* [`/v1/agent/force-leave/<node>`](#agent_force_leave)>: 强行删除节点
* [`/v1/agent/check/register`](#agent_check_register) : 注册新的本地检查
* [`/v1/agent/check/deregister/<checkID>`](#agent_check_deregister) : 注销本地检查
* [`/v1/agent/check/pass/<checkID>`](#agent_check_pass) : 标记本地检查为通过
* [`/v1/agent/check/warn/<checkID>`](#agent_check_warn) : 标记本地检查为警告
* [`/v1/agent/check/fail/<checkID>`](#agent_check_fail) : 标记本地检查为严重
* [`/v1/agent/check/update/<checkID>`](#agent_check_update) : 更新本地检查
* [`/v1/agent/service/register`](#agent_service_register) : 注册新的本地服务
* [`/v1/agent/service/deregister/<serviceID>`](#agent_service_deregister) : 注销本地服务
* [`/v1/agent/service/maintenance/<serviceID>`](#agent_service_maintenance) : 管理服务维护方式

### <a name="agent_checks"></a> /v1/agent/checks

这个端点用来返回在本地agent注册的所有检查。这些检查要么通过配置文件提供，要么使用HTTP API动态添加。注意，非常重要，**agent知道的检查可能和catalog报告的检查有所不同**。这通常是因为修改发生在没有leader被选举时。agent执行主动 [反熵/anti-entropy](../../docs/internals/anti-entropy.html)，因此在大多数情况下一切都将在几秒钟内被同步。

这个端点使用GET并返回类似这样的JSON body：

```javascript
{
  "service:redis": {
    "Node": "foobar",
    "CheckID": "service:redis",
    "Name": "Service 'redis' check",
    "Status": "passing",
    "Notes": "",
    "Output": "",
    "ServiceID": "redis",
    "ServiceName": "redis"
  }
}
```

> 注：为了得到类似上面这种排版整理的JSON内容，请在访问这个HTTP端点时增加pretty参数，例如：
> http://192.168.21.171:8500/v1/agent/self?pretty=1
> 否则默认是最小化的JSON格式，对人眼不友好

### <a name="agent_services"></a> /v1/agent/services

这个端点用来返回在本地agent注册的所有服务。这些服务要么通过配置文件提供，要么使用HTTP API动态添加。注意，非常重要，**agent知道的服务可能和catalog报告的服务有所不同**。这通常是因为修改发生在没有leader被选举时。agent执行主动 [反熵/anti-entropy](../../docs/internals/anti-entropy.html)，因此在大多数情况下一切都将在几秒钟内被同步。

这个端点使用GET并返回类似这样的JSON body：

```javascript
{
  "redis": {
    "ID": "redis",
    "Service": "redis",
    "Tags": null,
    "Address": "",
    "Port": 8000
  }
}
```

### <a name="agent_members"></a> /v1/agent/members

这个端点用来返回agent在集群gossip池中见到的成员。由于gossip的天性，这是最终一致：结果因agent而不同。节点的强一致性视图由"/v1/catalog/nodes"提供。

对于运行于server模式的agent，可以通过提供一个"?wan=1"查询参数来返回WAN成员列表，而不是默认返回LAN成员。

这个端点使用GET并返回类似这样的JSON body：

```javascript
[
  {
    "Name": "foobar",
    "Addr": "10.1.10.12",
    "Port": 8301,
    "Tags": {
      "bootstrap": "1",
      "dc": "dc1",
      "port": "8300",
      "role": "consul"
    },
    "Status": 1,
    "ProtocolMin": 1,
    "ProtocolMax": 2,
    "ProtocolCur": 2,
    "DelegateMin": 1,
    "DelegateMax": 3,
    "DelegateCur": 3
  }
]
```

### <a name="agent_self"></a> /v1/agent/self

这个端点用于返回本地agent的配置和成员信息。

返回类似这样的JSON body：

```javascript
{
  "Config": {
    "Bootstrap": true,
    "Server": true,
    "Datacenter": "dc1",
    "DataDir": "/tmp/consul",
    "DNSRecursor": "",
    "DNSRecursors": [],
    "Domain": "consul.",
    "LogLevel": "INFO",
    "NodeName": "foobar",
    "ClientAddr": "127.0.0.1",
    "BindAddr": "0.0.0.0",
    "AdvertiseAddr": "10.1.10.12",
    "Ports": {
      "DNS": 8600,
      "HTTP": 8500,
      "RPC": 8400,
      "SerfLan": 8301,
      "SerfWan": 8302,
      "Server": 8300
    },
    "LeaveOnTerm": false,
    "SkipLeaveOnInt": false,
    "StatsiteAddr": "",
    "Protocol": 1,
    "EnableDebug": false,
    "VerifyIncoming": false,
    "VerifyOutgoing": false,
    "CAFile": "",
    "CertFile": "",
    "KeyFile": "",
    "StartJoin": [],
    "UiDir": "",
    "PidFile": "",
    "EnableSyslog": false,
    "RejoinAfterLeave": false
  },
  "Coord": {
    "Adjustment": 0,
    "Error": 1.5,
    "Vec": [0,0,0,0,0,0,0,0]
  },
  "Member": {
    "Name": "foobar",
    "Addr": "10.1.10.12",
    "Port": 8301,
    "Tags": {
      "bootstrap": "1",
      "dc": "dc1",
      "port": "8300",
      "role": "consul",
      "vsn": "1",
      "vsn_max": "1",
      "vsn_min": "1"
    },
    "Status": 1,
    "ProtocolMin": 1,
    "ProtocolMax": 2,
    "ProtocolCur": 2,
    "DelegateMin": 2,
    "DelegateMax": 4,
    "DelegateCur": 4
  }
}
```

### <a name="agent_maintenance"></a> /v1/agent/maintenance

节点维护端点可以将agent设置为"维护模式"。在维护模式期间，这个节点将被标记为不可到达并不会出现在DNS或者API查询中。这个API调用是幂等的。维护模式是持久化的，并且将在agent重启时自动恢复。

`?enable`标记是必须的。可接受的值要不是`true` (进入维护模式) or `false` (恢复正常操作).

`?reason`标记是可选的。如果提供，它的值应该是一个文本字符串，解释将节点设置为维护模式的原因。这只是简单的帮助人类操作者。如果没有提供原因，将使用默认值。

成功时的返回码是200。

### <a name="agent_join"></a> /v1/agent/join/\<address\>

这个端点使用GET访问并用于通知agent去试图连接到给定地址。对于运行在服务器模式的agent，提供"?wan=1"查询参数会导致agent试图使用WAN池加入。

成功时的返回码是200。

### <a name="agent_force_leave"></a> /v1/agent/force-leave/\<node\>

这个端点使用GET访问并用于通知agent强制让节点进入`left/离开` 状态。如果节点意外失败，那么它将在`failed`状态。一旦在`failed`状态，consul将试图重连，而属于这个节点的服务和检查将不会被清楚。强行让节点进入`left`状态容许它的老的记录被删除。

这个端点总是返回200.

### <a name="agent_check_register"></a> /v1/agent/check/register

这个register端点用于添加一个新的检查到本地agent。有更多关于检查 [](/docs/agent/checks.html) 有更多的文档There is more documentation on checks [here](/docs/agent/checks.html).
The register endpoint is used to add a new check to the local agent.
There is more documentation on checks [here](/docs/agent/checks.html).
Checks may be of script, HTTP, TCP, or TTL type. The agent is responsible for
managing the status of the check and keeping the Catalog in sync.

The register endpoint expects a JSON request body to be PUT. The request
body must look like:

```javascript
{
  "ID": "mem",
  "Name": "Memory utilization",
  "Notes": "Ensure we don't oversubscribe memory",
  "Script": "/usr/local/bin/check_mem.py",
  "DockerContainerID": "f972c95ebf0e",
  "Shell": "/bin/bash",
  "HTTP": "http://example.com",
  "TCP": "example.com:22",
  "Interval": "10s",
  "TTL": "15s"
}
```

The `Name` field is mandatory, as is one of `Script`, `HTTP`, `TCP` or `TTL`.
`Script`, `TCP` and `HTTP` also require that `Interval` be set.

If an `ID` is not provided, it is set to `Name`. You cannot have duplicate
`ID` entries per agent, so it may be necessary to provide an `ID`.

The `Notes` field is not used internally by Consul and is meant to be human-readable.

If a `Script` is provided, the check type is a script, and Consul will
evaluate the script every `Interval` to update the status.

If a `DockerContainerID` is provided, the check is a Docker check, and Consul will
evaluate the script every `Interval` in the given container using the specified
`Shell`. Note that `Shell` is currently only supported for Docker checks.

An `HTTP` check will perform an HTTP GET request against the value of `HTTP` (expected to
be a URL) every `Interval`. If the response is any `2xx` code, the check is `passing`.
If the response is `429 Too Many Requests`, the check is `warning`. Otherwise, the check
is `critical`.

An `TCP` check will perform an TCP connection attempt against the value of `TCP`
(expected to be an IP/hostname and port combination) every `Interval`.  If the
connection attempt is successful, the check is `passing`.  If the connection
attempt is unsuccessful, the check is `critical`.  In the case of a hostname
that resolves to both IPv4 and IPv6 addresses, an attempt will be made to both
addresses, and the first successful connection attempt will result in a
successful check.

If a `TTL` type is used, then the TTL update endpoint must be used periodically to update
the state of the check.

The `ServiceID` field can be provided to associate the registered check with an
existing service provided by the agent.

The `Status` field can be provided to specify the initial state of the health
check.

This endpoint supports [ACL tokens](/docs/internals/acl.html). If the query
string includes a `?token=<token-id>`, the registration will use the provided
token to authorize the request. The token is also persisted in the agent's
local configuration to enable periodic
[anti-entropy](/docs/internals/anti-entropy.html) syncs and seamless agent
restarts.

The return code is 200 on success.

### <a name="agent_check_deregister"></a> /v1/agent/check/deregister/\<checkId\>

This endpoint is used to remove a check from the local agent.
The `CheckID` must be passed on the path. The agent will take care
of deregistering the check from the Catalog.

The return code is 200 on success.

### <a name="agent_check_pass"></a> /v1/agent/check/pass/\<checkId\>

This endpoint is used with a check that is of the [TTL type](/docs/agent/checks.html).
When this endpoint is accessed via a GET, the status of the check is set to `passing`
and the TTL clock is reset.

The optional "?note=" query parameter can be used to associate a human-readable message
with the status of the check. This will be passed through to the check's `Output` field
in the check endpoints.

The return code is 200 on success.

### <a name="agent_check_warn"></a> /v1/agent/check/warn/\<checkId\>

This endpoint is used with a check that is of the [TTL type](/docs/agent/checks.html).
When this endpoint is accessed via a GET, the status of the check is set to `warning`,
and the TTL clock is reset.

The optional "?note=" query parameter can be used to associate a human-readable message
with the status of the check. This will be passed through to the check's `Output` field
in the check endpoints.

The return code is 200 on success.

### <a name="agent_check_fail"></a> /v1/agent/check/fail/\<checkId\>

This endpoint is used with a check that is of the [TTL type](/docs/agent/checks.html).
When this endpoint is accessed via a GET, the status of the check is set to `critical`,
and the TTL clock is reset.

The optional "?note=" query parameter can be used to associate a human-readable message
with the status of the check. This will be passed through to the check's `Output` field
in the check endpoints.

The return code is 200 on success.

### <a name="agent_check_update"></a> /v1/agent/check/update/\<checkId\>

This endpoint is used with a check that is of the [TTL type](/docs/agent/checks.html).
When this endpoint is accessed with a PUT, the status and output of the check are
updated and the TTL clock is reset.

This endpoint expects a JSON request body to be put. The request body must look like:

```javascript
{
  "Status": "passing",
  "Output": "curl reported a failure:\n\n..."
}
```

The `Status` field is mandatory, and must be set to "passing", "warning", or "critical".

`Output` is an optional field that will associate a human-readable message with the status
of the check, such as the output of the checking script or process. This will be truncated
if it exceeds 4KB in size. This will be passed through to the check's `Output` field in the
check endpoints.

The return code is 200 on success.

### <a name="agent_service_register"></a> /v1/agent/service/register

The register endpoint is used to add a new service, with an optional health check,
to the local agent. There is more documentation on services [here](/docs/agent/services.html).
The agent is responsible for managing the status of its local services, and for sending updates
about its local services to the servers to keep the global Catalog in sync.

The register endpoint expects a JSON request body to be PUT. The request
body must look like:

```javascript
{
  "ID": "redis1",
  "Name": "redis",
  "Tags": [
    "master",
    "v1"
  ],
  "Address": "127.0.0.1",
  "Port": 8000,
  "Check": {
    "Script": "/usr/local/bin/check_redis.py",
    "HTTP": "http://localhost:5000/health",
    "Interval": "10s",
    "TTL": "15s"
  }
}
```

The `Name` field is mandatory,  If an `ID` is not provided, it is set to `Name`.
You cannot have duplicate `ID` entries per agent, so it may be necessary to provide an ID
in the case of a collision.

`Tags`, `Address`, `Port` and `Check` are optional.

`Address` will default to that of the agent if not provided.

If `Check` is provided, only one of `Script`, `HTTP`, or `TTL` should be specified.
`Script` and `HTTP` also require `Interval`. The created check will be named "service:\<ServiceId\>".
There is more information about checks [here](/docs/agent/checks.html).

This endpoint supports [ACL tokens](/docs/internals/acl.html). If the query
string includes a `?token=<token-id>`, the registration will use the provided
token to authorize the request. The token is also persisted in the agent's
local configuration to enable periodic
[anti-entropy](/docs/internals/anti-entropy.html) syncs and seamless agent
restarts.

The return code is 200 on success.

### <a name="agent_service_deregister"></a> /v1/agent/service/deregister/\<serviceId\>

The deregister endpoint is used to remove a service from the local agent.
The ServiceID must be passed after the slash. The agent will take care
of deregistering the service with the Catalog. If there is an associated
check, that is also deregistered.

The return code is 200 on success.

### <a name="agent_service_maintenance"></a> /v1/agent/service/maintenance/\<serviceId\>

The service maintenance endpoint allows placing a given service into
"maintenance mode". During maintenance mode, the service will be marked as
unavailable and will not be present in DNS or API queries. This API call is
idempotent. Maintenance mode is persistent and will be automatically restored
on agent restart.

The `?enable` flag is required.  Acceptable values are either `true` (to enter
maintenance mode) or `false` (to resume normal operation).

The `?reason` flag is optional.  If provided, its value should be a text string
explaining the reason for placing the service into maintenance mode. This is simply
to aid human operators. If no reason is provided, a default value will be used instead.

The return code is 200 on success.

