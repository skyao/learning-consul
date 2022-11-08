---
title: "Linux安装"
linkTitle: "Linux"
weight: 10
date: 2021-01-29
description: >
  在linxu上安装配置consul
---



>  参考官方文档： https://developer.hashicorp.com/consul/tutorials/get-started-vms/virtual-machine-gs-deploy



Consul是一个服务网络解决方案，使你能够管理服务之间以及跨内部和多云环境和运行时间的安全网络连接。Consul提供服务发现、服务网、流量管理和网络基础设施设备的自动更新。

在本教程中，你将在一个虚拟机（VM）上配置、部署和启动一个Consul服务器。部署完Consul后，你将使用用户界面、CLI和API与Consul进行交互。

在下面的教程中，你将部署一个演示应用程序，并将其配置为使用Consul的服务发现和服务网格功能。在这个过程中，你将学习如何利用Consul来安全地连接你在任何环境下运行的服务。

## 搭建单个集群

### 前提条件

- 在vm上安装 consul 二进制文件

  从 https://developer.hashicorp.com/consul/downloads 页面下载 consul 的 linunx_amd64 版本：

  https://releases.hashicorp.com/consul/1.13.3/consul_1.13.3_linux_amd64.zip

  解压缩之后将 consul 二进制放到 `~/work/soft/consul`目录下，将 consul 加入到path中: 

  ```bash
  # consul
  export PATH=$PATH:/home/sky/work/soft/consul
  ```

  检查consul 版本:

  ```bash
  $ consul version    
  Consul v1.13.3
  Revision b29e5894
  Build Date 2022-10-19T19:49:59Z
  Protocol 2 spoken by default, understands 2 to 3 (agent will automatically use protocol >2 when speaking to compatible agents)
  ```

- git

- curl,jq,dig

  ```bash
  sudo apt install curl jq
  ```

### 生成consul服务器配置

为了安全地配置Consul，确保Consul服务器和客户之间的所有通信都不能被非预期的代理访问，你需要向Consul提供：

- 一个 gossip 加密密钥。
- 一个来自私有CA的根证书授权（CA）证书。Consul将使用该CA证书来签署Consul数据中心的所有证书。
- 用于你打算部署的每台服务器的证书密钥对，由上述CA签署。

此外，你要启用ACL，以确保对你的Consul数据中心的每个请求都得到授权。

本教程和交互式实验环境使用教程的GitHub资源库中的脚本来生成这些秘密和Consul配置文件。你将使用这些脚本来生成你的Consul服务器配置。

在个人环境中，clone 下面的 learn-consul-get-started-vms 仓库：

```bash
cd soft/consul
git clone https://github.com/hashicorp-education/learn-consul-get-started-vms.git
```

创建一个名为 default 的默认集群配置，日常开发用，就用自己的用户名：

```bash
export DATACENTER="dc1" \
export DOMAIN="consul" \
export CONSUL_DATA_DIR="/home/sky/work/soft/consul/default/data" \
export CONSUL_CONFIG_DIR="/home/sky/work/soft/consul/default/config"
```

创建配置和运行 consul server agent 的所有必须的文件：

```bash
 $./scripts/generate_consul_server_config.sh  

Clean existing configuration
Generate Consul folders
Generate agent configuration - agent-server-secure.hcl
Generate server configuration - agent-server-specific.hcl
Generate gossip encryption key configuration - agent-gossip-encryption.hcl
Create CA for Consul datacenter
==> Saved consul-agent-ca.pem
==> Saved consul-agent-ca-key.pem
Create server Certificate and key pair
==> WARNING: Server Certificates grants authority to become a
    server and access all state in the cluster including root keys
    and all ACL tokens. Do not distribute them to production hosts
    that are not server nodes. Store them as securely as CA keys.
==> Using consul-agent-ca.pem and consul-agent-ca-key.pem
==> Saved dc1-server-consul-0.pem
==> Saved dc1-server-consul-0-key.pem
Generate TLS configuration - agent-server-tls.hcl
Generate ACL configuration - agent-server-acl.hcl
Validate configuration
Config validation failed: Multiple private IPv4 addresses found. Please configure one with 'bind' and/or 'advertise'.
Configuration invalid. Exiting.
```

错误提示：有多个IPv4地址。检查 `scripts/generate_consul_server_config.sh ` 文件可以看到最后执行检验的命令是：

```bash
$ consul validate ${CONSUL_CONFIG_DIR}
Config validation failed: Multiple private IPv4 addresses found. Please configure one with 'bind' and/or 'advertise'.
```

此时生成的配置文件存放在前面制定的 `/home/sky/work/soft/consul/default/config` 目录下：

```bash
$  config pwd
/home/sky/work/soft/consul/default/config
$  config ls
agent-gossip-encryption.hcl  agent-server-specific.hcl  consul-agent-ca.pem
agent-server-acl.hcl         agent-server-tls.hcl       dc1-server-consul-0-key.pem
agent-server-secure.hcl      consul-agent-ca-key.pem    dc1-server-consul-0.pem
```

修改 `agent-server-secure.hcl ` 文件，在address 下面加入 bind_addr 和 advertise_addr 设置：

```json
# Addresses and ports
addresses {
  grpc = "127.0.0.1"
  https = "0.0.0.0"
  dns = "0.0.0.0"
}

# add if there are multiple ipv4 address
bind_addr = "192.168.0.10"
advertise_addr = "192.168.0.10"
```

也可以后续在启动 consul agent 时手工绑定某个IP地址，比如 `consul agent -bind 192.168.0.10`

#### 配置文件

脚本生成了多个配置文件，将配置分开，这样更容易阅读和调整它们，以适应环境。下面是生成的文件和它们的作用描述。

- `agent-gossip-encryption.hcl` 文件配置 gossip 加密。
- `agent-server-acl.hcl` 文件配置 ACL 系统。
- `agent-server-secure.hcl` 文件配置了安全服务器代理。这是配置示例，不应该在生产环境中使用。
- `agent-server-specific.hcl` 文件包含特定的服务器参数。
- `agent-server-tls.hcl` 文件配置了特定的TLS加密。
- `*.pem` 文件是证书密钥对，Consul用来执行数据中心通信的mTLS。

###  启动 consul 服务器

```bash
consul agent -node=consul -config-dir=${CONSUL_CONFIG_DIR} -data-dir=${CONSUL_DATA_DIR} 
```

报错:

```bash
==> Error loading from /etc/consul/config/consul-agent-ca.pem: open /etc/consul/config/consul-agent-ca.pem: no such file or directory
```

打开生成的配置文件  `agent-server-tls.hcl`

```properties
## TLS Encryption (requires cert files to be present on the server nodes)
ca_file   = "/etc/consul/config/consul-agent-ca.pem"
cert_file = "/home/sky/work/soft/consul/default/config/dc1-server-consul-0.pem"
key_file  = "/home/sky/work/soft/consul/default/config/dc1-server-consul-0-key.pem"
```

这里的 ca_file 有问题，修改为使用  CONSUL_CONFIG_DIR :

```properties
## TLS Encryption (requires cert files to be present on the server nodes)
#ca_file   = "/etc/consul/config/consul-agent-ca.pem"   
ca_file   = "/home/sky/work/soft/consul/default/config/consul-agent-ca.pem"
cert_file = "/home/sky/work/soft/consul/default/config/dc1-server-consul-0.pem"
key_file  = "/home/sky/work/soft/consul/default/config/dc1-server-consul-0-key.pem"
```

重新执行:

```bash
consul agent -node=consul -config-dir=${CONSUL_CONFIG_DIR} -data-dir=${CONSUL_DATA_DIR} 
```

就可以正常启动了：

```bash
==> Starting Consul agent...
              Version: '1.13.3'
           Build Date: '2022-10-19 19:49:59 +0000 UTC'
              Node ID: '2e015547-e8dc-66ba-e5b4-6d85d7360d68'
            Node name: 'consul'
           Datacenter: 'dc1' (Segment: '<all>')
               Server: true (Bootstrap: true)
          Client Addr: [127.0.0.1] (HTTP: 8500, HTTPS: 8443, gRPC: 8502, DNS: 8600)
         Cluster Addr: 192.168.0.10 (LAN: 8301, WAN: 8302)
    Gossip Encryption: true
     Auto-Encrypt-TLS: true
            HTTPS TLS: Verify Incoming: false, Verify Outgoing: true, Min Version: TLSv1_2
             gRPC TLS: Verify Incoming: false, Min Version: TLSv1_2
     Internal RPC TLS: Verify Incoming: true, Verify Outgoing: true (Verify Hostname: true), Min Version: TLSv1_2
```

> 备注：提交了一个 issue https://github.com/consul/consul/issues/5026 

为方便使用，添加 alias

```bash
alias start-consul='consul agent -node=consul -config-dir="/home/sky/work/soft/consul/default/config" -data-dir="/home/sky/work/soft/consul/default/data"'
```

### 配置 consul CLI

```bash
consul acl bootstrap --format json > /home/sky/work/soft/consul/default/acl-token-bootstrap.json
```

> 备注：注意这个命令只能跑一次，因此要小心保存生成的acl文件。

执行下面命令获取一下SecretID：

```bash
cat "${CONSUL_CONFIG_DIR}"/../acl-token-bootstrap.json | jq -r ".SecretID"

5919d75e-41a5-aa0c-480a-e26863c0ff55
```

记下这个 SecretID，后面登录 consul UI 页面时需要用到。

配置环境变量（前四个是之前配置的，这里也要用到），注意修改 CONSUL_HTTP_ADDR ：

```bash
export DATACENTER="dc1" \
export DOMAIN="consul" \
export CONSUL_DATA_DIR="/home/sky/work/soft/consul/default/data" \
export CONSUL_CONFIG_DIR="/home/sky/work/soft/consul/default/config" \
export CONSUL_HTTP_ADDR="https://192.168.0.10:8443" \
export CONSUL_HTTP_SSL=true \
export CONSUL_CACERT="${CONSUL_CONFIG_DIR}/consul-agent-ca.pem" \
export CONSUL_TLS_SERVER_NAME="server.${DATACENTER}.${DOMAIN}" \
export CONSUL_FQDN_ADDR="consul" \
export CONSUL_HTTP_TOKEN=`cat "${CONSUL_CONFIG_DIR}"/../acl-token-bootstrap.json | jq -r ".SecretID"`
```





执行 consul info 命令检验一下：

```bash
consul info                                                                                     
agent:
	check_monitors = 0
	check_ttls = 0
	checks = 0
	services = 0
build:
	prerelease = 
	revision = b29e5894
	version = 1.13.3
	version_metadata = 
consul:
	acl = enabled
	bootstrap = true
	known_datacenters = 1
	leader = true
	leader_addr = 192.168.0.10:8300
	server = true
raft:
	applied_index = 21
	commit_index = 21
	fsm_pending = 0
	last_contact = 0
	last_log_index = 21
	last_log_term = 2
	last_snapshot_index = 0
	last_snapshot_term = 0
	latest_configuration = [{Suffrage:Voter ID:138d8891-d164-e6c0-48e1-9ac5232eb83c Address:192.168.0.10:8300}]
	latest_configuration_index = 0
	num_peers = 0
	protocol_version = 3
	protocol_version_max = 3
	protocol_version_min = 0
	snapshot_version_max = 1
	snapshot_version_min = 0
	state = Leader
	term = 2
runtime:
	arch = amd64
	cpu_count = 24
	goroutines = 117
	max_procs = 24
	os = linux
	version = go1.18.1
serf_lan:
	coordinate_resets = 0
	encrypted = true
	event_queue = 1
	event_time = 2
	failed = 0
	health_score = 0
	intent_queue = 0
	left = 0
	member_time = 1
	members = 1
	query_queue = 0
	query_time = 1
serf_wan:
	coordinate_resets = 0
	encrypted = true
	event_queue = 0
	event_time = 1
	failed = 0
	health_score = 0
	intent_queue = 0
	left = 0
	member_time = 1
	members = 1
	query_queue = 0
	query_time = 1
```

### 创建服务器端令牌

暂时先不设置，应该不影响测试



### 和 consul 服务器交互

用命令行查看consul成员：

```bash
$ consul members
Build   Protocol  DC   Partition  Segment
consul  192.168.0.10:8301  alive   server  1.13.3  2         dc1  default    <all>
```

也可以打开UI页面 `https://192.168.0.10:8443/`, 登录时使用前面得到的 SecretID

方便期间，设置环境变量的事情也交给 alias 吧：

```bash
alias set_consul='export DATACENTER=dc1;export DOMAIN=consul;export CONSUL_DATA_DIR="/home/sky/work/soft/consul/default/data ";export CONSUL_CONFIG_DIR="/home/sky/work/soft/consul/default/config";export CONSUL_HTTP_ADDR="https://192.168.0.10:8443" ;exportCONSUL_HTTP_SSL=true;export CONSUL_CACERT="${CONSUL_CONFIG_DIR}/consul-agent-ca.pem";export CONSUL_TLS_SERVER_NAME="server.${DATACENTER}.${DOMAIN}";export CONSUL_FQDN_ADDR="consul";export CONSUL_HTTP_TOKEN=5919d75e-41a5-aa0c-480a-e26863c0ff55'
```



## 搭建多个测试用的集群

为了方便测试跨集群访问的功能，需要在当前机器上额外搭建多个consul集群

### 搭建集群2

准备目录并设置：

```bash
export DATACENTER="dc1" \
export DOMAIN="consul" \
export CONSUL_DATA_DIR="/home/sky/work/soft/consul/cluster2/data" \
export CONSUL_CONFIG_DIR="/home/sky/work/soft/consul/cluster2/config"
```

执行:

```bash
./scripts/generate_consul_server_config.sh
```

记得修改 agent-server-tls.hcl 中错误的 ca_file 设置

```bash
ca_file   = "/home/sky/work/soft/consul/cluster2/config/consul-agent-ca.pem"
```

修改 agent-server-secure.hcl 文件，在所有端口前加2：

```properties
ports {
  grpc  = 28502
  http  = 28500
  https = 28443
  dns   = 28600
  server = 28300
}
# add if there are multiple ipv4 address
bind_addr = "192.168.0.10"
advertise_addr = "192.168.0.10"
```

启动：

```bash
consul agent -node=consul -config-dir=${CONSUL_CONFIG_DIR} -data-dir=${CONSUL_DATA_DIR} -serf-wan-port=28302 -serf-lan-port=28301
```

> 备注：serf-wan-port 和 serf-lan-port 不知道怎么在配置文件中设置，只好在命令行中输入。

还是加个 alias 吧：

```bash
alias start-consul2='consul agent -node=consul -config-dir="/home/sky/work/soft/consul/cluster2/config" -data-dir="/home/sky/work/soft/consul/cluster2/data" -serf-wan-port=28302 -serf-lan-port=28301'
```

启动客户端前配置环境变量（前四个是之前配置的，这里也要用到），注意修改 CONSUL_HTTP_ADDR ：

```bash
export DATACENTER="dc1" \
export DOMAIN="consul" \
export CONSUL_DATA_DIR="/home/sky/work/soft/consul/cluster2/data" \
export CONSUL_CONFIG_DIR="/home/sky/work/soft/consul/cluster2/config" \
export CONSUL_HTTP_ADDR="https://192.168.0.10:28443" \
export CONSUL_HTTP_SSL=true \
export CONSUL_CACERT="${CONSUL_CONFIG_DIR}/consul-agent-ca.pem" \
export CONSUL_TLS_SERVER_NAME="server.${DATACENTER}.${DOMAIN}" \
export CONSUL_FQDN_ADDR="consul"
```

生成acl：

```bash
consul acl bootstrap --format json > /home/sky/work/soft/consul/cluster2/acl-token-bootstrap.json
```

执行下面命令获取一下SecretID：

```bash
cat "${CONSUL_CONFIG_DIR}"/../acl-token-bootstrap.json | jq -r ".SecretID"

63ceeceb-d3f2-b7b2-039c-fbd2ed986b21
```

记下这个 SecretID，后面登录 consul UI 页面时需要用到。

```bash
export CONSUL_HTTP_TOKEN=`cat "${CONSUL_CONFIG_DIR}"/../acl-token-bootstrap.json | jq -r ".SecretID"`
```

方便起见，设置环境变量的事情交给 alias 吧：

```bash
alias set_consul2='export DATACENTER=dc1;export DOMAIN=consul;export CONSUL_DATA_DIR="/home/sky/work/soft/consul/cluster2/data ";export CONSUL_CONFIG_DIR="/home/sky/work/soft/consul/cluster2/config";export CONSUL_HTTP_ADDR="https://192.168.0.10:28443" ;exportCONSUL_HTTP_SSL=true;export CONSUL_CACERT="${CONSUL_CONFIG_DIR}/consul-agent-ca.pem";export CONSUL_TLS_SERVER_NAME="server.${DATACENTER}.${DOMAIN}";export CONSUL_FQDN_ADDR="consul";export CONSUL_HTTP_TOKEN=63ceeceb-d3f2-b7b2-039c-fbd2ed986b21'
```

检验：

```bash
consul info
consul members
```

打开UI页面： `https://192.168.0.10:28443/`



