---
title: 云原生实战 - Devops
created: '2022-05-13T03:19:58.803Z'
modified: '2023-06-18T02:58:03.785Z'
---

# 云原生实战 - Devops

最近微服务框架需要部署k8s，看了国内KubeSphere开源容器平台的文档觉得简单实用，于是就在本地的 Linux 上以 All-in-One 模式安装 KubeSphere测试使用。难免会遇到了一些问题就记录下来。

### 使用devops功能模块

参考文档编辑流水线第一步`clone code`，指定容器和添加嵌套步骤，在git clone 码云的公开仓库时始终报错

```shell
Cloning the remote Git repository
Cloning repository https://gitee.com/xinliangnote/go-gin-api.git
 > git init /home/jenkins/agent/workspace/mall-devops5fg5w/demo # timeout=10
Fetching upstream changes from https://gitee.com/xinliangnote/go-gin-api.git
 > git --version # timeout=10
 > git --version # 'git version 2.11.0'
using GIT_ASKPASS to set credentials 
 > git fetch --tags --progress -- https://gitee.com/xinliangnote/go-gin-api.git +refs/heads/*:refs/remotes/origin/* # timeout=10
ERROR: Error cloning remote repo 'origin'
hudson.plugins.git.GitException: Command "git fetch --tags --progress -- https://gitee.com/xinliangnote/go-gin-api.git +refs/heads/*:refs/remotes/origin/*" returned status code 128:
stdout: 
stderr: fatal: unable to access 'https://gitee.com/xinliangnote/go-gin-api.git/': Failed to connect to gitee.com port 443: Connection refused
```

##### 1.本地宿主机器测试`ping` gitee.com

```shell
root@gary-asus:~# ping -c 4 gitee.com
PING fn0wz54v.dayugslb.com (212.64.62.183) 56(84) bytes of data.
64 字节，来自 212.64.62.183 (212.64.62.183): icmp_seq=1 ttl=52 时间=41.7 毫秒
64 字节，来自 212.64.62.183 (212.64.62.183): icmp_seq=2 ttl=52 时间=41.6 毫秒
64 字节，来自 212.64.62.183 (212.64.62.183): icmp_seq=3 ttl=52 时间=42.9 毫秒
64 字节，来自 212.64.62.183 (212.64.62.183): icmp_seq=4 ttl=52 时间=41.2 毫秒

--- fn0wz54v.dayugslb.com ping 统计 ---
已发送 4 个包， 已接收 4 个包, 0% 包丢失, 耗时 3004 毫秒
rtt min/avg/max/mdev = 41.198/41.826/42.880/0.632 ms
```

##### 2.测试docker容器环境ping gitee.com

将kubesphere jenkins流水线`clone code`阶段的任务修改为shell任务`ping -c 4 gitee.com`得到运行结果：

```shell
+ ping -c 4 gitee.com
PING gitee.com.Home (127.0.0.1) 56(84) bytes of data.
64 bytes from localhost (127.0.0.1): icmp_seq=1 ttl=64 time=0.072 ms
64 bytes from localhost (127.0.0.1): icmp_seq=2 ttl=64 time=0.058 ms
64 bytes from localhost (127.0.0.1): icmp_seq=3 ttl=64 time=0.079 ms
64 bytes from localhost (127.0.0.1): icmp_seq=4 ttl=64 time=0.061 ms

--- gitee.com.Home ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3075ms
rtt min/avg/max/mdev = 0.058/0.067/0.079/0.011 ms
```

两次`ping`结果不一致，本地宿主机器测试是正确的，流水线docker环境目标服务器ip结果是错误的`127.0.0.1`。搜索一番资料总结问题是本地服务器dns解析错误的原因。

```shell
root@gary-asus:~# cat /etc/resolv.conf 
# operation for /etc/resolv.conf.
nameserver 127.0.0.53
options edns0 trust-ad
search Home
```

增加谷歌dns服务器

```shell
root@gary-asus:~# cat /etc/resolv.conf 
# See man:systemd-resolved.service(8) for details about the supported modes of
# operation for /etc/resolv.conf.

nameserver 8.8.8.8
nameserver 8.8.4.4
nameserver 127.0.0.53
options edns0 trust-ad
search Home
```

重新运行流水线，`clone code`阶段成功。

```bash
The recommended git tool is: NONE
No credentials specified
Warning: JENKINS-30600: special launcher org.csanchez.jenkins.plugins.kubernetes.pipeline.ContainerExecDecorator$1@70ebc0b2; decorates RemoteLauncher[hudson.remoting.Channel@2e9eba6e:JNLP4-connect connection from 10.233.73.240/10.233.73.240:50548] will be ignored (a typical symptom is the Git executable not being run inside a designated container)
Cloning the remote Git repository
Cloning repository https://gitee.com/xinliangnote/go-gin-api.git
 > git init /home/jenkins/agent/workspace/mall-devops5fg5w/demo # timeout=10
Fetching upstream changes from https://gitee.com/xinliangnote/go-gin-api.git
 > git --version # timeout=10
 > git --version # 'git version 2.11.0'
 > git fetch --tags --progress -- https://gitee.com/xinliangnote/go-gin-api.git +refs/heads/*:refs/remotes/origin/* # timeout=10
Avoid second fetch
 > git config remote.origin.url https://gitee.com/xinliangnote/go-gin-api.git # timeout=10
 > git config --add remote.origin.fetch +refs/heads/*:refs/remotes/origin/* # timeout=10
 > git rev-parse refs/remotes/origin/master^{commit} # timeout=10
Checking out Revision 4c37a7e6b571b29b91c59a129c52cea8e899e7a2 (refs/remotes/origin/master)
 > git config core.sparsecheckout # timeout=10
 > git checkout -f 4c37a7e6b571b29b91c59a129c52cea8e899e7a2 # timeout=10
 > git branch -a -v --no-abbrev # timeout=10
 > git checkout -b master 4c37a7e6b571b29b91c59a129c52cea8e899e7a2 # timeout=10
Commit message: "feature(1.2.8): swagger 接口文档新增 Security"
First time build. Skipping changelog.
```



联想到上一次测试“企业空间”》“应用管理”》“应用仓库”，添加第三方仓库验证失败也是一直提示类似的错误：

```bash
Bad Request
Get "https://charts.bitnami.com/bitnami/index.yaml": dial tcp 127.0.0.1:443: connect: connection refused
```



### Ubuntu 20.04设置DNS解析（解决resolve.conf被覆盖问题）

配置需要的域名解析服务器，需要在/etc/resolve.conf添加例如：“nameserver 8.8.8.8”，但是这个文件即使我们修改了，很快又会被覆盖，而且我们注意一个细节——文件的开头就注明了“Do not edit”，直接修改/etc/resolve.conf是不正确的。



```shell
# This file is managed by man:systemd-resolved(8). Do not edit.
```

说明这个文件是被systemd-resolved这个服务托管的

需要修改`/etc/systemd/resolved.conf`这个配置文件

然后重启服务 `systemctl restart systemd-resolved.service`

```shell
#  This file is part of systemd.
#
#  systemd is free software; you can redistribute it and/or modify it
#  under the terms of the GNU Lesser General Public License as published by
#  the Free Software Foundation; either version 2.1 of the License, or
#  (at your option) any later version.
#
# Entries in this file show the compile time defaults.
# You can change settings by editing this file.
# Defaults can be restored by simply deleting this file.
#
# See resolved.conf(5) for details

[Resolve]
#DNS=
DNS=8.8.8.8
#FallbackDNS=
#Domains=
#LLMNR=no
#MulticastDNS=no
#DNSSEC=no
#DNSOverTLS=no
#Cache=no-negative
#DNSStubListener=yes
#ReadEtcHosts=yes
```
