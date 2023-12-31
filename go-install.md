---
title: go-install
created: '2022-03-30T01:50:37.552Z'
modified: '2023-12-01T02:11:19.496Z'
---

# go-install

```shell
sudo apt update
sudo apt install build-essential

wget https://studygolang.com/dl/golang/go1.16.15.linux-amd64.tar.gz
rm -rf /usr/local/go && tar -C /usr/local -xzf go1.16.15.linux-amd64.tar.gz

wget https://studygolang.com/dl/golang/go1.17.8.linux-amd64.tar.gz
rm -rf /usr/local/go && tar -C /usr/local -xzf go1.17.8.linux-amd64.tar.gz

# gobin
# vim /etc/profile
# export PATH=$PATH:/usr/local/go/bin
# source /etc/profile

# gobin gopath
vim ~/.profile 
# 或者
vim ~/.bashrc 
export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin
source ~/.profile

go env -w GO111MODULE=on
go env -w GOPROXY=https://goproxy.cn,direct
```

在Ubuntu中有如下几个文件可以设置环境变量
1. `/etc/profile` 
在登录时操作系统定制用户环境时使用的第一个文件,此文件为系统的每个用户设置环境信息,当用户第一次登录时该文件被执行.

2. `/etc/environment` 
在登录时操作系统使用的第二个文件,系统在读取你自己的`profile`前设置环境文件的环境变量.

3. `~/.profile`
在登录时用到的第三个文件是`.profile`文件,每个用户都可使用该文件输入专用于自己使用的`shell`信息,当用户登录时该文件仅仅执行一次!

4. `/etc/bash.bashrc`
为每一个运行`bash shell`的用户执行此文件,当`bash shell`被打开时该文件被读取.

5. `~/.bashrc`
该文件包含专用于你的`bash shell`的`bash`信息,当登录时以及每次打开新的shell时该文件被读取.

