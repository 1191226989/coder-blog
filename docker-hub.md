---
title: docker-hub
created: '2022-05-25T05:21:42.629Z'
modified: '2023-12-01T02:57:10.174Z'
---

# docker-hub

### Login

```bash
docker login -u [username] -p [password] [registry]
docker logout
```

### Push

```bash
docker tag local-image:tagname new-repo:tagname
docker push new-repo:tagname
```

### Delete

```bash
docker image rm image-name:tagname
docker rmi image-name:tagname
docker rmi new-repo:tagname
```

### Mysql

```bash
docker run -p 3306:3306 --name mysql-01 \
-v /mydata/mysql/log:/var/log/mysql \
-v /mydata/mysql/data:/var/lib/mysql \
-v /mydata/mysql/conf:/etc/mysql/conf.d \
-e MYSQL_ROOT_PASSWORD=root \
--restart=always \
-d mysql:5.7 
```

### Redis

```bash
#创建配置文件
mkdir -p /mydata/redis/conf && vim /mydata/redis/conf/redis.conf

#配置示例
appendonly yes
port 6379
bind 0.0.0.0


#docker启动redis
docker run -d -p 6379:6379 --restart=always \
-v /mydata/redis/conf/redis.conf:/etc/redis/redis.conf \
-v  /mydata/redis-01/data:/data \
 --name redis-01 redis:6.2.5 \
 redis-server /etc/redis/redis.conf
```

### Etcd

```bash
$ docker run -d --name etcd-server \
    --publish 2379:2379 \
    --publish 2380:2380 \
    --env ALLOW_NONE_AUTHENTICATION=yes \
    --env ETCD_ADVERTISE_CLIENT_URLS=http://etcd-server:2379 \
    bitnami/etcd:latest
```

