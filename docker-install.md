---
title: docker-install
created: '2022-03-30T01:50:37.552Z'
modified: '2023-12-01T03:08:01.226Z'
---

# docker-install

```shell
sudo apt-get remove docker docker-engine docker.io containerd runc

sudo apt-get update

sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
    
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update

sudo apt-get install docker-ce docker-ce-cli containerd.io

sudo apt-get -y install docker-compose
docker-compose -v

# docker-compose编排后更新了配置文件
docker-compose build
docker-compose build [service]
docker-compose up -d
docker-compose --compatibility up
docker-compose up --force-recreate -d
docker-compose logs [service]
# 用以清理不再使用的docker镜像
docker image prune
```
```
sudo docker run -p 3306:3306 --name mysql -e MYSQL_ROOT_PASSWORD=123456 -d mysql:5.7
 
duso docker run -p 3306:3306 --name mysql \
-v /usr/local/docker/mysql/conf:/etc/mysql \
-v /usr/local/docker/mysql/logs:/var/log/mysql \
-v /usr/local/docker/mysql/data:/var/lib/mysql \
-e MYSQL_ROOT_PASSWORD=123456 \
-d mysql:5.7
 
docker run -d --name redis -p 6379:6379 redis:latest --requirepass 123456 
 
docker exec -it redis /bin/bash
 
docker run -p 6379:6379 --name redis -v /usr/local/docker/redis.conf:/etc/redis/redis.conf -v /usr/local/docker/data:/data -d redis redis-server /etc/redis/redis.conf --appendonly yes
```
- 参数解释
`-p 6379:6379` 端口映射 [主机端口]:[容器端口]
`--name myredis` 指定该容器名称
`-v` 挂载目录 [主机目录]:[容器目录]
`-d redis` 表示后台启动redis
`redis-server /etc/redis/redis.conf` 以配置文件启动redis，加载容器内的conf文件，最终找到的是挂载的目录/usr/local/docker/redis.conf
`appendonly yes` 开启redis 持久化
`--restart=always` 在容器退出时总是重启容器,主要应用在生产环境
`--privileged=true` 特权模式,授权容器与主机相同的权限
 
```sh
docker pull bitnami/etcd:latest
docker network create app-tier --driver bridge
docker run -d --name etcd-server \
    --network app-tier \
    --publish 2379:2379 \
    --publish 2380:2380 \
    --env ALLOW_NONE_AUTHENTICATION=yes \
    --env ETCD_ADVERTISE_CLIENT_URLS=http://etcd-server:2379 \
    bitnami/etcd:latest
docker run -it --rm \
    --network app-tier \
    --env ALLOW_NONE_AUTHENTICATION=yes \
    bitnami/etcd:latest etcdctl --endpoints http://etcd-server:2379 put /message Hello
```
    
