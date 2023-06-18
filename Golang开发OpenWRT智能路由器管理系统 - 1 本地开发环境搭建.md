---
title: Golang开发OpenWRT智能路由器管理系统 - 1 本地开发环境搭建
created: '2023-06-15T14:36:48.083Z'
modified: '2023-06-16T11:32:48.348Z'
---

# Golang开发OpenWRT智能路由器管理系统 - 1 本地开发环境搭建

反复几次试过https://openwrt.org/文档提供的docker搭建测试环境方法，存在缺少部分命令和ubusd无法启动的问题

使用VirtualBox 搭建 OpenWRT 本地开发环境
https://openwrt.org/zh/docs/guide-user/virtualization/virtualbox-vm

1. 下载编译完好的镜像文件
https://downloads.openwrt.org/releases/22.03.5/targets/x86/64/openwrt-22.03.5-x86-64-generic-ext4-combined.img.gz

2. 解压缩并提取 OpenWRT .img文件
openwrt-22.03.5-x86-64-generic-ext4-combined.img

3. 将 OpenWRT .img文件转换为virtualbox VDI（需要将virtualbox的安装目录加到环境变量）
```shell
vboxmanage.exe convertfromraw  --format VDI openwrt-22.03.5-x86-64-generic-ext4-combined.img  openwrt.vdi
```

4. 调整 VDI 文件的大小，增加虚拟磁盘容量的大小。
```shell
vboxmanage.exe modifyhd --resize 512 openwrt.vdi
```

5. 选择 使用已有的虚拟硬盘文件(Use an existing hard disk file)新建虚拟机，类型选择 Linux，版本选择Linux 2.6/3.x/4.x(64-bit)

6. 网卡的设置可以参考开发文档，如果是桥接网卡需要虚拟机系统ip调整为宿主机同一个网段
```shell
# 编辑网络配置
vim /etc/config/network
# LAN 口参考配置：（请根据实际网络配置网段，这里以192.168.88.0 网段为例，虚拟 OpenWrt 管理地址设置为静态：192.168.88.1）
config interface 'lan'
        option ifname 'eth0'
        option proto 'static'
        option ipaddr '192.168.88.1'
        option netmask '255.255.255.0'

# 需要关闭虚拟机的firewall宿主机才能ssh连接
service firewall stop
# 重新启动网络服务
service network restart

#浏览器打开 192.168.88.1 就能进入路由器管理页面

```
```shell
# 更新 openwrt 软件包
opkg update
# 安装 luci web ui
opkg install luci

/etc/init.d/uhttpd enable
/etc/init.d/uhttpd start			#（启动http服务，使用NAT端口转发）
/etc/init.d/firewall stop			#（关闭防火墙）
# 重启OpenWrt
reboot now
```

- 安装完成以后启动你的虚拟机
- 等待4秒，GRUB程序将会自动引导系统
- 当启动信息停止滚动后，按下 回车键Enter 来激活命令行

7. 笔记本电脑通过虚拟机OpenWrt上网
- 这里配置为“笔记本——虚拟机OpenWrt——路由器”。如需配置旁路由的方式上网，请自行查找相关教程。

- 打开“更改适配器选项”，右击“以太网”打开属性，取消勾选“Internet协议版本4(TCP/IPv4)”和“Internet协议版本6(TCP/IPv6)”，点击确定。

- 右击“VirtualBox Host-Only Ethernet Adapter”打开属性，打开“Internet协议版本4(TCP/IPv4)”，默认网关和DNS都设置为OpenWrt的“lan”网卡地址192.168.88.1，点击确定即可完成设置。

8. 接下来就是需要用goang开发一个类似luci web ui的路由器管理系统

#### 遇到的问题
- openwrt虚拟机不能连接外网

在网上博客也看到很多人遇到类似的问题，试了一些方法也没有解决问题。还是找到开源组织网站的文档才解决了不能连接外网的问题。`https://openwrt.org/zh/docs/guide-user/virtualization/virtualbox-vm`

按照文档提供的说明重新装了一次就成功了。文档强调要开启两个虚拟网卡，这个是解决问题的关键，也是和其他普通linux虚拟机不同的地方。如果以前只开启了一个虚拟网卡，就先关闭虚拟机然后在虚拟机 设置》网络设置 选项里将网卡1设置为 `仅主机(Host-only)网络`，将网卡2设置为`(网络地址转换)NAT`，然后重新启动虚拟机就可以正常连接外网了。

如果遇到宿主机无法ping虚拟机的情况，可以将网卡1改为`桥接网卡(Bridged Adapter)`，或者再增加一个网卡3并设置为`桥接网卡(Bridged Adapter)`，并将openwrt虚拟机`/etc/config/network`配置文件的`lan`配置项的ip改为和宿主机同一个网段的ip，就可以解决问题了。






