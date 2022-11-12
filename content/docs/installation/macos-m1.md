---
title: "MacOS M1安装"
linkTitle: "MacOS M1"
weight: 20
date: 2021-01-29
description: >
  在 MacOS M1上安装配置consul
---



>  参考官方文档： https://developer.hashicorp.com/consul/tutorials/get-started-vms/virtual-machine-gs-deploy 和在linux上的安装。





## 搭建单个集群

### 前提条件

- 在vm上安装 consul 二进制文件

  从 https://developer.hashicorp.com/consul/downloads 页面下载 consul 的 linunx_amd64 版本：

  https://releases.hashicorp.com/consul/1.13.3/consul_1.13.3_darwin_arm64.zip

  解压缩之后将 consul 二进制放到 `~/work/soft/consul`目录下，将 consul 加入到path中: 

  ```bash
  # consul
  export PATH=$PATH:/Users/sky/work/soft/consul
  ```

  检查consul 版本:

  ```bash
  $ consul --version
  Consul v1.13.3
  Revision b29e5894
  Build Date 2022-10-19T19:49:59Z
  Protocol 2 spoken by default, understands 2 to 3 (agent will automatically use protocol >2 when speaking to compatible agents)
  ```

- git

- curl,jq,dig

  ```bash
  brew install curl jq
  ```

### 生成consul服务器配置

clone 下面的 learn-consul-get-started-vms 仓库：

```bash
cd work/soft/consul
git clone https://github.com/hashicorp-education/learn-consul-get-started-vms.git
```

创建一个名为 default 的默认集群配置，日常开发用，就用自己的用户名：

```bash
export DATACENTER="dc1" \
export DOMAIN="consul" \
export CONSUL_DATA_DIR="/Users/sky/work/soft/consul/default/data" \
export CONSUL_CONFIG_DIR="/Users/sky/work/soft/consul/default/config"
```

创建配置和运行 consul server agent 的所有必须的文件：

```bash
 # 确保 CONSUL_DATA_DIR 和 CONSUL_CONFIG_DIR 目录已经创建好
 
 $./scripts/generate_consul_server_config.sh  

......
Configuration invalid. Exiting.
```

#### 配置文件

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
cert_file = "/Users/sky/work/soft/consul/default/config/dc1-server-consul-0.pem"
key_file  = "/Users/sky/work/soft/consul/default/config/dc1-server-consul-0-key.pem"
```

这里的 ca_file 有问题，修改为使用  CONSUL_CONFIG_DIR :

```properties
## TLS Encryption (requires cert files to be present on the server nodes)
#ca_file   = "/etc/consul/config/consul-agent-ca.pem"   
ca_file   = "/Users/sky/work/soft/consul/default/config/consul-agent-ca.pem"
cert_file = "/Users/sky/work/soft/consul/default/config/dc1-server-consul-0.pem"
key_file  = "/Users/sky/work/soft/consul/default/config/dc1-server-consul-0-key.pem"
```

重新执行:

```bash
consul agent -node=consul -config-dir=${CONSUL_CONFIG_DIR} -data-dir=${CONSUL_DATA_DIR} 
```

为方便使用，添加 alias

```bash
alias start-consul='consul agent -node=consul -config-dir="/Users/sky/work/soft/consul/default/config" -data-dir="/Users/sky/work/soft/consul/default/data"'
```

### 配置 consul CLI

打开另一个终端：

```bash
# set ENV first
export DATACENTER="dc1" \
export DOMAIN="consul" \
export CONSUL_DATA_DIR="/Users/sky/work/soft/consul/default/data" \
export CONSUL_CONFIG_DIR="/Users/sky/work/soft/consul/default/config"

consul acl bootstrap --format json > /Users/sky/work/soft/consul/default/acl-token-bootstrap.json
```

> 备注：注意这个命令只能跑一次，因此要小心保存生成的acl文件。

执行下面命令获取一下SecretID：

```bash
cat "${CONSUL_CONFIG_DIR}"/../acl-token-bootstrap.json | jq -r ".SecretID"

0b8ca5d0-aace-9d33-cf13-62059fd39573
```

记下这个 SecretID，后面登录 consul UI 页面时需要用到。

配置环境变量（前四个是之前配置的，这里也要用到），注意修改 CONSUL_HTTP_ADDR ：

```bash
export DATACENTER="dc1" \
export DOMAIN="consul" \
export CONSUL_DATA_DIR="/Users/sky/work/soft/consul/default/data" \
export CONSUL_CONFIG_DIR="/Users/sky/work/soft/consul/default/config" \
export CONSUL_HTTP_ADDR="https://127.0.0.1:8443" \
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
alias set_consul='export DATACENTER=dc1;export DOMAIN=consul;export CONSUL_DATA_DIR="/Users/sky/work/soft/consul/default/data ";export CONSUL_CONFIG_DIR="/Users/sky/work/soft/consul/default/config";export CONSUL_HTTP_ADDR="https://127.0.0.1:8443" ;exportCONSUL_HTTP_SSL=true;export CONSUL_CACERT="${CONSUL_CONFIG_DIR}/consul-agent-ca.pem";export CONSUL_TLS_SERVER_NAME="server.${DATACENTER}.${DOMAIN}";export CONSUL_FQDN_ADDR="consul";export CONSUL_HTTP_TOKEN=0b8ca5d0-aace-9d33-cf13-62059fd39573'
```



## 搭建多个测试用的集群

为了方便测试跨集群访问的功能，需要在当前机器上额外搭建多个consul集群

### 搭建集群2

准备目录并设置：

```bash
export DATACENTER="dc1" \
export DOMAIN="consul" \
export CONSUL_DATA_DIR="/Users/sky/work/soft/consul/cluster2/data" \
export CONSUL_CONFIG_DIR="/Users/sky/work/soft/consul/cluster2/config"
```

执行:

```bash
./scripts/generate_consul_server_config.sh
```

记得修改 agent-server-tls.hcl 中错误的 ca_file 设置

```bash
ca_file   = "/Users/sky/work/soft/consul/cluster2/config/consul-agent-ca.pem"
```

修改 agent-server-secure.hcl 文件，在所有端口前加2：

```properties
ports {
  grpc  = 28502
  http  = 28500
  https = 28443
  dns   = 28600
  server = 28300 # 多加一行
}
```

启动：

```bash
consul agent -node=consul -config-dir=${CONSUL_CONFIG_DIR} -data-dir=${CONSUL_DATA_DIR} -serf-wan-port=28302 -serf-lan-port=28301
```

> 备注：serf-wan-port 和 serf-lan-port 不知道怎么在配置文件中设置，只好在命令行中输入。

还是加个 alias 吧：

```bash
alias start-consul2='consul agent -node=consul -config-dir="/Users/sky/work/soft/consul/cluster2/config" -data-dir="/Users/sky/work/soft/consul/cluster2/data" -serf-wan-port=28302 -serf-lan-port=28301'
```

启动客户端前配置环境变量（前四个是之前配置的，这里也要用到），注意修改 CONSUL_HTTP_ADDR ：

```bash
export DATACENTER="dc1" \
export DOMAIN="consul" \
export CONSUL_DATA_DIR="/Users/sky/work/soft/consul/cluster2/data" \
export CONSUL_CONFIG_DIR="/Users/sky/work/soft/consul/cluster2/config" \
export CONSUL_HTTP_ADDR="https://127.0.0.1:28443" \
export CONSUL_HTTP_SSL=true \
export CONSUL_CACERT="${CONSUL_CONFIG_DIR}/consul-agent-ca.pem" \
export CONSUL_TLS_SERVER_NAME="server.${DATACENTER}.${DOMAIN}" \
export CONSUL_FQDN_ADDR="consul"
```

生成acl：

```bash
consul acl bootstrap --format json > /Users/sky/work/soft/consul/cluster2/acl-token-bootstrap.json
```

执行下面命令获取一下SecretID：

```bash
cat "${CONSUL_CONFIG_DIR}"/../acl-token-bootstrap.json | jq -r ".SecretID"

6c0b9c61-c642-99db-9e58-972d5b838b9b
```

记下这个 SecretID，后面登录 consul UI 页面时需要用到。

```bash
export CONSUL_HTTP_TOKEN=`cat "${CONSUL_CONFIG_DIR}"/../acl-token-bootstrap.json | jq -r ".SecretID"`
```

方便起见，设置环境变量的事情交给 alias 吧：

```bash
alias set_consul2='export DATACENTER=dc1;export DOMAIN=consul;export CONSUL_DATA_DIR="/Users/sky/work/soft/consul/cluster2/data ";export CONSUL_CONFIG_DIR="/Users/sky/work/soft/consul/cluster2/config";export CONSUL_HTTP_ADDR="https://127.0.0.1:28443" ;exportCONSUL_HTTP_SSL=true;export CONSUL_CACERT="${CONSUL_CONFIG_DIR}/consul-agent-ca.pem";export CONSUL_TLS_SERVER_NAME="server.${DATACENTER}.${DOMAIN}";export CONSUL_FQDN_ADDR="consul";export CONSUL_HTTP_TOKEN=6c0b9c61-c642-99db-9e58-972d5b838b9b'
```

检验：

```bash
consul info
consul members
```

打开UI页面： `https://127.0.0.1:28443/`，用上面保存的 SecretID 登录。





