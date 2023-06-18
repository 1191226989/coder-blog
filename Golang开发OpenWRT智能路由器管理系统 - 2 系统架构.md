---
title: Golang开发OpenWRT智能路由器管理系统 - 2 系统架构
created: '2023-06-15T15:13:35.938Z'
modified: '2023-06-17T05:14:42.172Z'
---

# Golang开发OpenWRT智能路由器管理系统 - 2 系统架构

golang开发一个类似luci web ui的路由器管理系统

#### 整体流程
##### 方案1

```shell

前端 ---> golang api ---> ubus ---> 修改Package服务配置 ---> 重启服务 

```
[https://openwrt.org/docs/guide-developer/ubus](https://openwrt.org/docs/guide-developer/ubus)

> Path only contains the first context. E.g. network for network.interface.wan

| path	| Description	| Package |
| ----  | ---- | ---- |
| dhcp |	dhcp server	| odhcpd	|
| file |file |	rpcd |
| hostapd |	acesspoints |	wpad/hostapd |
| iwinfo |	wireless informations |	rpcd iwinfo |
| log |	logging |	procd |
| mdns |	mdns avahi replacement |	mdnsd |
| network |	network |	netifd |
| service |	init/service |	procd |
| session |	Session management |	rpcd |
| system |	system misc |	procd |
| uci |	Unified Configuration Interface |	rpcd |

- 有些`Package`服务`ubus`提供了可以命令行直接调用的方法文档（rpcd/netifd/procd）
- 有些`Package`服务`ubus`没有提供可以命令行直接调用的方法文档（odhcpd/wpad/hostapd/mdnsd），但是`rpcd`服务提供了的`uci`修改对应`package`配置的方法。[https://openwrt.org/docs/guide-developer/ubus/uci]
- 使用`ubus`提供的方法，开发人员只需要调用相关命令，传参接收返回结果
- golang可以通过`os.exec`包调用`ubus`提供的命令行（shell）
- 个别服务模块的配置不支持`uci`修改。
[https://openwrt.org/docs/guide-user/base-system/notuci.config]

##### 方案2

```shell
前端 ---> golang api ---> 修改Package服务配置文件（uci） ---> 重启服务 
```
- 该方案直接修改各个`Package`服务模块的配置文件，这些配置文件都是支持uci的格式。
- 部分业务功能逻辑需要自行写代码实现（管理员登录/修改密码/退出登录等），当然也可以调用方案1的`ubus`方法来实现。
- golang的扩展包[https://github.com/digineo/go-uci]


