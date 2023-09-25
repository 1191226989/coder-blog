---
title: kubesphere流水线代码部署git问题
created: '2023-09-25T04:33:07.892Z'
modified: '2023-09-25T04:49:42.384Z'
---

# kubesphere流水线代码部署git问题

https://www.kubesphere.io/zh/
Linux上以 All-in-One 模式安装 KubeSphere 测试使用

#### 使用 devops 功能模块代码部署问题
编辑流水线第一步 `clone code`，指定容器和添加嵌套步骤，在 git clone 码云的公开仓库时始终报错
```sh
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
1. 本地宿主机器测试 `ping gitee.com`结果正确
```sh
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
2. 测试 docker 容器环境 `ping gitee.com`结果错误
将 kubesphere jenkins 流水线 `clone code` 阶段的任务修改为 `shell` 任务 `ping -c 4 gitee.com` 得到运行结果：
```sh
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
docker 环境gitee.com目标服务器 ip 结果是错误的 `127.0.0.1`。原因是本地服务器 dns 服务器错误。
```sh
root@gary-asus:~# cat /etc/resolv.conf 
# operation for /etc/resolv.conf.
nameserver 127.0.0.53
options edns0 trust-ad
search Home
```

3. 增加谷歌 dns 服务器
```sh
root@gary-asus:~# cat /etc/resolv.conf 
# See man:systemd-resolved.service(8) for details about the supported modes of
# operation for /etc/resolv.conf.

nameserver 8.8.8.8
nameserver 8.8.4.4
nameserver 127.0.0.53
options edns0 trust-ad
search Home
```
重新运行流水线，`clone code` 阶段成功
```sh
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

#### 应用仓库添加第三方仓库验证失败问题
企业空间”》“应用管理”》“应用仓库”，添加第三方仓库验证失败提示错误：
```sh
Bad Request
Get "https://charts.bitnami.com/bitnami/index.yaml": dial tcp 127.0.0.1:443: connect: connection refused
```
这个也是dns配置错误无法解析的问题


