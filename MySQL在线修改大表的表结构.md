---
title: MySQL在线修改大表的表结构
created: '2023-06-22T01:06:28.863Z'
modified: '2023-06-22T01:13:54.011Z'
---

# MySQL在线修改大表的表结构

当一个表中的数据量很大的时候，对表中的列的字段类型进行修改时会锁表，从而影响业务。

- 在线修改大表的结构执行时间不可预估，一般时间较长
- 修改表结构是表级锁，影响数据写入
- 长时间修改表结构，如果中途失败，由于修改表结构是一个事务，失败后会还原表结构，整个过程中表都是加锁不可写入
- 修改大表结构容易导致服务器CPU、IO消耗上升
- 容易导致主从延时，影响业务读取

##### 方案一 : 从表修改，主从切换
现在从服务器上修改，然后主从切换。 切换完以后在此修改新的从服务器。 需要主从切换

##### 方案二：pt-online-schema-change
https://www.percona.com/downloads
```
pt-online-schema-change --alter="modify product varchar(100) not null default ''" --user=root --password=artisan D=artisan,t=t_order  --charset=utf8 --execute
```
如果有从库需要加参数 `--nocheck-replication-filters`

```
--user=  // 连接mysql的用户名
--password= // 连接mysql的密码
--host= // 连接mysql的服务器
P=3306  // 连接mysql的端口号
D=  // 连接mysql的库名
t=  // 连接mysql的表名
--alter  // 修改表结构的语句，实际执行的alter语句， 多个更改用逗号分隔
--execute  // 执行修改表结构
--dry-run // 创建并修改新表，但不会建触发器拷贝数据
--charset=utf8  // 使用utf8编码
--no-version-check   // 不检查版本，在阿里云的服务器一般加此参数
```

写个pt.sh复用剧本
```
#!/bin/bash
table=$1
alter_content=$2

conn_host= '127.0.0.1'
conn_user='user'
conn_pwd='password'
conn_db='databasename'

echo "$table"
echo "$alter_content"
/root/percona-toolkit-2.2.19/bin/pt-online-schema-change --charset=utf8 --no-version-check --user=${conn_user} --password=${conn_pwd} --host=${conn_host}  P=3306,D=${conn_db},t=$table --alter "${alter_conment}" --execute
```
```
sh pt.sh t_order "ADD COLUMN `request_id` varchar(32) NOT NULL DEFAULT '' "

sh pt.sh t_order "ADD INDEX idx_address(address)"
```
##### 使用限制
1. 默认如果检测到有复制过滤会拒绝改表，修改参数为--[no]check-replication-filters

2. 默认如果检测到主从复制延迟会自动停止数据拷贝，调节参数为--max-lag

3. 默认如果检测到服务器负载过重会停止或中断操作，调节参数为--max-load和--critical-load

4. 默认会设置锁等待超时时间为1s来避免干扰其他事务的进行，调节参数为--lock-wait-timeout

5. 默认如果检测到外键冲突后会拒绝改表，调节参数为--alter-foreign-keys-method


- 建议操作方法
1. 先执行--dry-run 检查系统支持情况
2. 执行 --execute

##### 工作原理

1. 创建一个和要执行 alter 操作的表一样的新的空表，后缀默认是new。

2. 在新表执行alter table 语句，因为是空表，执行速度很快。

3. 在原表中创建触发器3个触发器分别对应insert，update，delete操作。

4. 以一定块大小从原表拷贝数据到临时表，拷贝过程中通过原表上的触发器在原表进行的写操作都会更新到新建的临时表，注意这里是Replace操作。

5. 表明替换 将原表名table修改为 tableold, 将tablenew 表明修改为原表名tabl

6. 如果有关联该表的外键，根据alter-foreign-keys-method参数的值，检测外键相关的表，做相应设置的处理。

7. 默认最后将旧原表删除。


