---
title: Golang配置私有仓库依赖
created: '2023-10-04T07:10:11.164Z'
modified: '2023-10-17T13:41:32.213Z'
---

# Golang配置私有仓库依赖

linux 开发环境 golang 私有仓库依赖配置：

- golang 版本要求：1.14+
- go mod 配置：
```sh
go env -w GOPRIVATE="gitlab.xxx.com"    //配置私有仓库域名
go env -w GONOPROXY="gitlab.xxx.com"    //此配置下的域名默认不走代理
go env -w GONOSUMDB="gitlab.xxx.com"    //此配置下的域名默认不进行gosumdb校验
go env -w GOINSECURE="gitlab.xxx.com"    //此配置下的域名采用http协议。默认采用https，请根据实际情况进行配置。
```
- git 账户和密码： 私有仓库一般都需要进行登录，可以通过隐藏文件进行用户名及密码配置

文件路径：
```
~/.netrc    //Linux 系统
```
文件内容：
```
machine gitlab.xxx.com //域名:gitlab.xxx.com
login xxx //账号
password xxx //密码
```
