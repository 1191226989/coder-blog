---
title: Golang开发OpenWRT智能路由器-3扩容根目录
created: '2023-09-11T09:43:21.202Z'
modified: '2023-09-11T10:23:59.193Z'
---

# Golang开发OpenWRT智能路由器-3扩容根目录

openwrt系统安装所用的服务器磁盘空间很小，默认情况下磁盘会有一部分剩余的空间会被隐藏起来无法使用，需要自行迁移系统并重新挂载才能扩容使用。

此方法同样适用于virtualbox虚拟机搭建openwrt环境的根目录扩容

1. 远程登录openwrt服务器
```sh
ssh root@192.168.100.1
```

2. 安装cfdisk分区工具
```sh
opkg update
opkg install cfdisk
```

3. 使用cfdisk工具分区
创建新分区 `/dev/sda3`，然后格式化这个分区
```sh
mkfs.ext4 /dev/sda3
```

4. 在系统自带的Luci web界面增加挂载点
有些系统没有这个功能选项，需要自行安装后重启系统
```sh
opkg update
opkg install block-mount
```
- 依次点击 系统 》挂载点 找到并点击全局设置中的【生成配置】
- 在【挂载点】找到创建的新分区，点击 【修改】 重新调整挂载项目的设置
- 勾选“启用此挂载点”，挂载点选择为 “作为根文件系统使用”，完整复制根目录准备中的所有命令行后，点击【保存并应用】
- 服务器执行复制的命令行（可能需要将命令行的挂载目录改为`/dev/sda3`）
```sh
mkdir -p /tmp/introot
mkdir -p /tmp/extroot
mount --bind / /tmp/introot
mount /dev/sda3 /tmp/extroot
tar -C /tmp/introot -cvf - . | tar -C /tmp/extroot -xf -
umount /tmp/introot
umount /tmp/extroot
```
5. `reboot`重启openwrt服务器，检查根目录是否扩容成功
```sh
df -h 
```

