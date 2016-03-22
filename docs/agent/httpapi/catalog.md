目录 HTTP 端点
==============

> 注：内容翻译自官网文档 [CATALOG HTTP ENDPOINT](https://www.consul.io/docs/agent/http/catalog.html)

目录是用于注册和注销节点，服务和检查的端点。它也提供查询端点。

支持下列端点：

* [`/v1/catalog/register`](#catalog_register) : 注册新节点，服务或者检查
* [`/v1/catalog/deregister`](#catalog_deregister) : 注销新节点，服务或者检查
* [`/v1/catalog/datacenters`](#catalog_datacenters) : 列出已知的数据中心
* [`/v1/catalog/nodes`](#catalog_nodes) : 列出在给定数据中心中的节点
* [`/v1/catalog/services`](#catalog_services) : 列出在给定数据中心中的服务
* [`/v1/catalog/service/<service>`](#catalog_service) : 列出在给定服务中的节点
* [`/v1/catalog/node/<node>`](#catalog_nodes) : 列出节点提供的服务

`nodes` 和 `services` 端点支持阻塞查询和可调一致性模型。

### <a name="catalog_register"></a> /v1/catalog/register

register端点是在catalog中注册或者更新记录的低级别机制。注意：通常最好换用[agent endpoints](agent.html)来注册，因为他们更简单并进行了[anti-entropy](../../docs/internals/anti-entropy.html).

register 端点期待PUT的JSON request body。request body必须看上去像这样：

```javascript
{
  "Datacenter": "dc1",
  "Node": "foobar",
  "Address": "192.168.10.10",
  "Service": {
    "ID": "redis1",
    "Service": "redis",
    "Tags": [
      "master",
      "v1"
    ],
    "Address": "127.0.0.1",
    "TaggedAddresses": {
      "wan": "127.0.0.1"
    },
    "Port": 8000
  },
  "Check": {
    "Node": "foobar",
    "CheckID": "service:redis1",
    "Name": "Redis health check",
    "Notes": "Script based health check",
    "Status": "passing",
    "ServiceID": "redis1"
  }
}
```



