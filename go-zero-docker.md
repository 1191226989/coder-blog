---
title: go-zero-docker
created: '2022-04-01T01:58:29.198Z'
modified: '2023-06-18T03:03:59.451Z'
---

# go-zero-docker

`goctl docker` 可以极速生成一个 Dockerfile，帮助开发/运维人员加快部署节奏，降低部署复杂度。

- 选择最简单的镜像：比如alpine，整个镜像5M左右

- 设置镜像时区

  ```shell
  RUN apk add --no-cache tzdata
  ENV TZ Asia/Shanghai
  ```

  

### 多阶段构建

- 第一阶段构建出可执行文件，确保构建过程独立于宿主机
- 第二阶段将第一阶段的输出作为输入，构建出最终的极简镜像



### Dockerfile编写过程

在 greet 项目下创建一个 hello 服务

```shell
goctl api new hello
```

在 `hello` 目录下一键生成 `Dockerfile`

```shell
goctl docker -go hello.go
```

在 `hello` 目录或者其他目录下 `build` 镜像

```shell
docker build -t hello:v1 -f service/hello/Dockerfile .

docker build -t hello:v1 -f ./Dockerfile .

docker build -t hello:v1 .
```

查看镜像

```shell
REPOSITORY                     TAG       IMAGE ID       CREATED              SIZE
hello                          v1        21dd1da58414   27 seconds ago       16.5MB

```

启动服务

```shell
docker run --rm -it -p 8888:8888 hello:v1
```

测试服务

```shell
curl -i http://localhost:8888/from/you

HTTP/1.1 200 OK
Content-Type: application/json
Date: Thu, 10 Dec 2020 06:03:02 GMT
Content-Length: 14
{"message":""}
```



```shell
# 清除dangling images，指的是没有标签并且没有被容器使用的镜像
docker image prune

Total reclaimed space: 853.8MB
```



```shell
FROM golang:alpine AS builder

LABEL stage=gobuilder

ENV CGO_ENABLED 0
ENV GOPROXY https://goproxy.cn,direct

RUN apk update --no-cache && apk add --no-cache tzdata

WORKDIR /build

ADD go.mod .
ADD go.sum .
RUN go mod download
COPY . .
COPY hello/etc /app/etc
RUN go build -ldflags="-s -w" -o /app/hello hello/hello.go


FROM alpine

RUN apk update --no-cache && apk add --no-cache ca-certificates
COPY --from=builder /usr/share/zoneinfo/Asia/Shanghai /usr/share/zoneinfo/Asia/Shanghai
ENV TZ Asia/Shanghai

WORKDIR /app
COPY --from=builder /app/hello /app/hello
COPY --from=builder /app/etc /app/etc

CMD ["./hello", "-f", "etc/hello-api.yaml"]

```



