---
title: 云原生实战 - Helm
created: '2022-05-23T05:41:17.304Z'
modified: '2023-06-18T02:56:40.958Z'
---

# 云原生实战 - Helm

想成功和正确地使用Helm，需要以下前置条件。

1. 一个 Kubernetes 集群
2. 确定你安装版本的安全配置
3. 安装和配置Helm。



### 用二进制版本安装

每个Helm [版本](https://github.com/helm/helm/releases)都提供了各种操作系统的二进制版本，这些版本可以手动下载和安装。

1. 下载 [需要的版本](https://github.com/helm/helm/releases)
2. 解压(`tar -zxvf helm-v3.0.0-linux-amd64.tar.gz`)
3. 在解压目中找到`helm`程序，移动到需要的目录中(`mv linux-amd64/helm /usr/local/bin/helm`)

然后就可以执行客户端程序并 [添加稳定仓库](https://helm.sh/zh/docs/intro/quickstart/#初始化): `helm help`.

**注意** 针对Linux AMD64，Helm的自动测试只有在CircleCi构建和发布时才会执行。

### 使用helm安装harbor

helm install my-harbor ./harbor -n harbor

helm uninstall my-harbor ./harbor -n harbor
