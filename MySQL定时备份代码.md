---
title: MySQL定时备份代码
created: '2023-08-26T07:50:59.824Z'
modified: '2023-08-26T07:52:37.121Z'
---

# MySQL定时备份代码

```shell
#!/bin/sh
bak_path="/mnt/dbbak" # 数据库数据文件备份目录

date_now=`date +%Y%m%d` # 当前日期
date_dep=`date -d "-1 week" +%Y%m%d` # 7天前日期

db_name="test" # 数据库名称
db_user="test" # 数据库用户
db_pass="test" # 数据库密码

cd $bak_path
echo "================Backup database================" >> bak.log
if [ -d $date_dep ]; then
        echo "`date '+%F %H:%M:%S'` Remove deprecated folder $date_dep." >> bak.log
        rm -rf $date_dep # 删除7天前备份数据
fi

if [ ! -d $date_now ]; then
        mkdir $date_now # 创建当前日期备份数据文件夹
fi

cd $date_now
echo "`date '+%F %H:%M:%S'` Begin to backup $db_name." >> ../bak.log
# 使用mysqldump备份testdb1数据库实例并压缩成gzip格式，test为数据库登录用户名和密码
/usr/local/mysql/bin/mysqldump -u$db_user -p$db_pass $db_name|gzip > $db_name`date +%Y%m%d%H%M%S`.sql.gz
echo "`date '+%F %H:%M:%S'` Finish to backup $db_name." >> ../bak.log

echo "" >> ../bak.log
```
