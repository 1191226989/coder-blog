---
title: docker-hub
created: '2022-05-25T05:21:42.629Z'
modified: '2023-06-18T02:59:49.864Z'
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





kubesphere account

```bash
workerzhang
workerzhang@qq.com
活跃	platform-regular	2022-05-25 15:11:01	
	
projectzhang
projectzhang@qq.com
活跃	platform-regular	2022-05-25 15:05:12	

	
leaderzhang
leaderzhang@qq.com
活跃	platform-regular	2022-05-25 15:12:00	

	
managerzhang
managerzhang@test.com
活跃	workspaces-manager	2022-05-23 21:58:30	

	
bosszhang
xiaozhang@qq.com
活跃	users-manager	2022-05-25 15:13:30	
	
admin
admin@kubesphere.io
活跃	platform-admin	2022-05-23 22:06:11
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
## 1、准备redis配置文件内容
mkdir -p /mydata/redis/conf && vim /mydata/redis/conf/redis.conf


##配置示例
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

