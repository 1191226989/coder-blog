---
title: go-zero-install
created: '2022-03-30T01:50:37.552Z'
modified: '2023-06-18T03:02:33.088Z'
---

# go-zero-install

#### tree 命令部分常用命令
```
tree -a 显示所有
tree -d 只显示文档夹
tree -L n 显示项目的层级，n表示层级数，比如想要显示项目三层结构，可以用tree -L 3
tree -I pattern 用于过滤不想要显示的文档或者文档夹。比如你想要过滤项目中的 node_modules 文档夹，可以使用 tree -I "node_modules"
tree > tree.md 将项目结构输出到 tree.md 这个文档
tree -N 防止中文名乱码
```

```shell
# 安装 protoc
wget https://github.com/protocolbuffers/protobuf/releases/download/v3.14.0/protoc-3.14.0-linux-x86_64.zip
unzip protoc-3.14.0-linux-x86_64.zip
mv bin/protoc /usr/local/bin/

wget https://github.com/protocolbuffers/protobuf/releases/download/v3.19.4/protoc-3.19.4-linux-x86_64.zip
unzip protoc-3.19.4-linux-x86_64.zip
mv bin/protoc /usr/local/bin/

# 安装 protoc-gen-go
go get -u github.com/golang/protobuf/protoc-gen-go@v1.3.2

go install google.golang.org/protobuf/cmd/protoc-gen-go@latest

go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest

option go_package = "."

# 安装 goctl 工具
go get -u github.com/zeromicro/go-zero/tools/goctl

go install github.com/zeromicro/go-zero/tools/goctl@latest
```



```shell
单体服务demo
goctl api new greet
cd greet
go mod tidy
go run greet.go -f etc/greet-api.yaml

curl -i http://localhost:8888/from/you

HTTP/1.1 200 OK
Content-Type: application/json
X-Trace-Id: 2566bb59aecea1e755eacb78484bec86
Date: Wed, 30 Mar 2022 11:57:19 GMT
Content-Length: 27

{"message":"hello go-zero"}
```



```shell
微服务demo
# 安装 etcd, mysql, redis

api网关层
#创建工作目录 shorturl 和 shorturl/api
mkdir -p shorturl/api

# 在 shorturl 目录下执行 go mod init shorturl 初始化 go.mod
sudo chown -R gary shorturl

module shorturl

go 1.16

require (
	github.com/zeromicro/go-zero v1.3.1
	google.golang.org/grpc v1.45.0 // indirect
)

goctl api -o shorturl.api
# 生成代码
goctl api go -api shorturl.api -dir .

transform rpc层
# 在 shorturl 目录下创建 rpc 目录
# 在 rpc/transform 目录下编写 transform.proto 文件

goctl rpc template -o transform.proto

# 用 goctl 生成 rpc 代码，在 rpc/transform 目录下执行命令
goctl rpc proto -src transform.proto -dir .
或者
新版本golang
option go_package = "./proto3;
goctl rpc protoc transform.proto --go_out=. --go-grpc_out=. --zrpc_out=.

go run transform.go -f etc/transform.yaml 

export ETCDCTL_API=3 
etcdctl get transform.rpc --prefix 
返回结果
transform.rpc/7587861361303463432
127.0.0.1:8080


修改 API Gateway 代码调用 transform rpc 服务
# 修改配置文件 shorturl-api.yaml，增加如下内容

Transform:
  Etcd:
    Hosts:
      - localhost:2379
    Key: transform.rpc
    
# 修改 internal/config/config.go 如下，增加 transform 服务依赖
type Config struct {
  rest.RestConf
  Transform zrpc.RpcClientConf     // 手动代码
}

# 修改 internal/svc/servicecontext.go，如下：
通过 ServiceContext 在不同业务逻辑之间传递依赖

# 修改 internal/logic/expandlogic.go 里的 Expand 方法，如下
通过调用 transformer 的 Expand 方法实现短链恢复到 url

# 修改 internal/logic/shortenlogic.go，如下：
通过调用 transformer 的 Shorten 方法实现 url 到短链的变换


 定义数据库表结构，并生成 CRUD+cache 代码
 # shorturl 下创建 rpc/transform/model 目录：mkdir -p rpc/transform/model
 
 # 在 rpc/transform/model 目录下编写创建 shorturl 表的 sql 文件 shorturl.sql，如下：
CREATE TABLE `shorturl`
(
  `shorten` varchar(255) NOT NULL COMMENT 'shorten key',
  `url` varchar(255) NOT NULL COMMENT 'original url',
  PRIMARY KEY(`shorten`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

# 创建 DB 和 table
create database gozero;
source shorturl.sql;

# 在 rpc/transform/model 目录下执行如下命令生成 CRUD+cache 代码，-c 表示使用 redis cache
goctl model mysql ddl -c -src shorturl.sql -dir .


修改 shorten/expand rpc 代码调用 crud+cache 代码
# 修改 rpc/transform/etc/transform.yaml，增加如下内容：
DataSource: root:password@tcp(localhost:3306)/gozero
Table: shorturl
Cache:
  - Host: localhost:6379
可以使用多个 redis 作为 cache，支持 redis 单点或者 redis 集群

# 修改 rpc/transform/internal/config/config.go，如下：
type Config struct {
  zrpc.RpcServerConf
  DataSource string             // 手动代码
  Table      string             // 手动代码
  Cache      cache.CacheConf    // 手动代码
}
增加了 mysql 和 redis cache 配置

# 修改 rpc/transform/internal/svc/servicecontext.go，如下：

# 修改 rpc/transform/internal/logic/expandlogic.go，如下：

# 修改 rpc/transform/internal/logic/shortenlogic.go，如下：

# 启动rpc服务
go run rpc/transform/transform.go -f rpc/transform/etc/transform.yaml

# 启动api服务
go run api/shorturl.go -f api/etc/shorturl-api.yaml

# shorten api 调用
curl -i "http://localhost:8888/shorten?url=http://www.xiaoheiban.cn"

# expand api 调用
curl -i "http://localhost:8888/expand?shorten=f35b2a"


```



```
一个基于docker的go-zero运行环境 https://github.com/nivin-studio/gonivinck

1. 按需修改 .env 配置
# 设置时区
TZ=Asia/Shanghai
# 设置网络模式
NETWORKS_DRIVER=bridge


# PATHS ##########################################
# 宿主机上代码存放的目录路径
CODE_PATH_HOST=./code
# 宿主机上Mysql Reids数据存放的目录路径
DATA_PATH_HOST=./data


# MYSQL ##########################################
# Mysql 服务映射宿主机端口号，可在宿主机127.0.0.1:3306访问
MYSQL_PORT=3306
MYSQL_USERNAME=admin
MYSQL_PASSWORD=123456
MYSQL_ROOT_PASSWORD=123456

# Mysql 可视化管理用户名称，同 MYSQL_USERNAME
MYSQL_MANAGE_USERNAME=admin
# Mysql 可视化管理用户密码，同 MYSQL_PASSWORD
MYSQL_MANAGE_PASSWORD=123456
# Mysql 可视化管理ROOT用户密码，同 MYSQL_ROOT_PASSWORD
MYSQL_MANAGE_ROOT_PASSWORD=123456
# Mysql 服务地址
MYSQL_MANAGE_CONNECT_HOST=mysql
# Mysql 服务端口号
MYSQL_MANAGE_CONNECT_PORT=3306
# Mysql 可视化管理映射宿主机端口号，可在宿主机127.0.0.1:1000访问
MYSQL_MANAGE_PORT=1000


# REDIS ##########################################
# Redis 服务映射宿主机端口号，可在宿主机127.0.0.1:6379访问
REDIS_PORT=6379

# Redis 可视化管理用户名称
REDIS_MANAGE_USERNAME=admin
# Redis 可视化管理用户密码
REDIS_MANAGE_PASSWORD=123456
# Redis 服务地址
REDIS_MANAGE_CONNECT_HOST=redis
# Redis 服务端口号
REDIS_MANAGE_CONNECT_PORT=6379
# Redis 可视化管理映射宿主机端口号，可在宿主机127.0.0.1:2000访问
REDIS_MANAGE_PORT=2000


# ETCD ###########################################
# Etcd 服务映射宿主机端口号，可在宿主机127.0.0.1:2379访问
ETCD_PORT=2379


# PROMETHEUS #####################################
# Prometheus 服务映射宿主机端口号，可在宿主机127.0.0.1:9000访问
PROMETHEUS_PORT=9000


# GRAFANA ########################################
# Grafana 服务映射宿主机端口号，可在宿主机127.0.0.1:3000访问
GRAFANA_PORT=3000


# JAEGER #########################################
# Jaeger 服务映射宿主机端口号，可在宿主机127.0.0.1:5000访问
JAEGER_PORT=5000


# DTM #########################################
# DTM HTTP 协议端口号
DTM_HTTP_PORT=36789
# DTM gRPC 协议端口号
DTM_GRPC_PORT=36790

2.启动服务
启动全部服务 docker-compose up -d
按需启动部分服务 docker-compose up -d etcd golang mysql redis

3.运行代码
将项目代码放置 CODE_PATH_HOST 指定的本机目录，进入 golang 容器，运行项目代码。
docker exec -it golang_1 bash

```

