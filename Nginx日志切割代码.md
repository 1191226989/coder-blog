---
title: Nginx日志切割代码
created: '2023-08-26T07:54:34.341Z'
modified: '2023-08-26T07:55:33.127Z'
---

# Nginx日志切割代码

```sh
#!/bin/sh
bak_path="/mnt/logbak" # 日志文件备份目录

date_now=`date +%Y%m%d` # 当前日期
date_dep=`date -d "-1 week" +%Y%m%d` # 7天前日期

host_name="test" #定义虚拟主机的目录名
logs_path="/mnt/logs/nginx" # 日志文件目录

cd $bak_path
echo "================Backup logs================" >> bak.log
if [ -d $date_dep ]; then
        echo "`date '+%F %H:%M:%S'` Remove deprecated folder $date_dep." >> bak.log
        rm -rf $date_dep # 删除7天前备份数据
fi

if [ ! -d $date_now ]; then
        mkdir $date_now # 创建当前日期备份数据文件夹
fi

cd $date_now
echo "`date '+%F %H:%M:%S'` Begin to backup logs." >> ../bak.log

mv ${logs_path}/${host_name}.access.log ${host_name}`date +%Y%m%d%H%M%S`.access.log
/usr/sbin/nginx -s reload 

echo "`date '+%F %H:%M:%S'` Finish to backup logs." >> ../bak.log

echo "" >> ../bak.log
```
