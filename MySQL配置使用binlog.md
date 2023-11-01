---
title: MySQL配置使用binlog
created: '2023-10-04T08:10:36.847Z'
modified: '2023-11-01T02:16:28.651Z'
---

# MySQL配置使用binlog

`binlog` 即二进制日志，记录了引起或可能引起数据库改变事件，包括事件发生的时间、开始位置、结束位置等信息，`select`、`show` 等查询语句不会引起数据库改变，因此不会被记录在 binlog 中。

对于事务的执行，只有事务提交时才会一次写入 `binlog`，对于非事务操作则每次语句执行成功后都会直接写入 `binlog`。

基于 `binlog` 可以看到每一次对数据库的修改是在何时以何种方式执行的，从而可以实现对任意条操作的回滚。

mysql 的主从同步机制也是依赖 `binlog` 来实现的，`binlog` 让从库可以精准还原主库的每一个操作。

```sql
-- 查看是否开启 binlog 日志
show variables like 'log_bin';
```

配置文件开启 binlog 及相关配置：
```sh
[mysqld]
server_id = 1234
binlog_format = MIXED                 # binlog日志格式
log_bin = /data/mysql/mysql-bin.log   # binlog日志文件
expire_logs_days = 7                  # binlog过期清理时间
max_binlog_size = 1000m               # binlog每个日志文件大小，默认为 1G
binlog_cache_size = 4m                # binlog缓存大小
max_binlog_cache_size = 512m          # 最大binlog缓存大小
binlog_do_db = shop                   # 指定需要记录 binlog 的数据库
binlog_ignore_db = test               # 指定忽略记录 binlog 的数据库
```

#### 日志格式
- STATEMENT 模式（SBR）
默认格式。 在这个模式下只会记录可能引起数据变更的 sql 语句。

1. 优点 
这个模式下，因为没有记录实际的数据，所以日志量和 IO 都消耗很低，性能是最优的。

2. 缺点 
但有些操作并不是确定的，比如 `uuid ()` 函数会随机产生唯一标识，当依赖 binlog 回放时，该操作生成的数据与原数据必然是不同的，此时可能造成无法预料的后果。 由于所有的操作都依赖于先后顺序，所以像使用 `AUTO_INCREMENT` 生成主键 id 的 `insert` 方法、数据的恢复等都必须串行执行。

- ROW 模式（RBR）
在该模式下会记录每次操作的源数据与修改后的目标数据，而不会记录 sql 语句，从 mysql 5.6.2 版本开始，你可以通过在配置文件中指定 `binlog_rows_query_log_events` 配置项为 `0` 或 `1` 来决定是否同时记录 sql 语句。 但对于 `GRANT`，`REVOKE`，`SET PASSWORD` 等管理语句仍然是以 SBR 方式来进行记录的。

1. 优点 
主要优势在于可以绝对精准的还原，从而保证了数据的安全与可靠。 并且复制和数据恢复过程可以是并发进行的。

2. 缺点 
该模式最大的缺点在于 binlog 体积会非常大，对于修改记录多、字段长度大的操作来说，RBR 记录时性能消耗会很严重。由于数据是通过二进制方式记录，无法直观的看到 binlog 究竟记录了什么信息。

- MIXED 模式（MBR）
MIXED 模式是对上述两种模式的混合使用，对于绝大部分操作都使用 SBR 来进行 binlog 的记录，只有以下操作使用 RBR 来实现：
1. 表的存储引擎为 `NDB`
2. 使用了 `uuid()`、`user()`、`current_user()`、`found_rows()`、`row_count()`、`sysdate()` 等不确定函数（`now()` 函数仍然会以 SBR 方式记录）
3. 使用了 `insert delay` 语句
4. 使用了临时表

```sql
show master status --当前的日志文件、偏移量等信息

show binary logs

show master logs

SHOW BINLOG EVENTS [IN ‘log_name’] [FROM pos]
```

#### mysqlbinlog

mysqlbinlog [option] log-file1 log-file2…
```sh
-d, --database=name     #只查看指定数据库的日志操作
-o, --offset=           #展示的起始偏移
-r, --result-file=name  #将输出的日志信息输出到指定的文件中，使用重定向也一样可以。
-s, --short-form=       #显示简单格式的日志，只记录一些普通的语句，会省略掉一些额外的信息如位置信息和时间信息以及基于行的日志。可以用来调试，生产环境不可使用
--set-charset=char_name #在输出日志信息到文件中时，在文件第一行加上set names char_name
--start-datetime, --stop-datetime       #指定输出开始时间和结束时间内的所有日志信息
--start-position=, --stop-position=     #指定输出开始位置和结束位置内的所有日志信息
-v, -vv   #显示更详细信息，基于row的日志默认不会显示出来，此时使用-v或-vv可以查看
```
#### 清理 binlog
```sh
# 删除所有 binlog，并让日志文件从 000001 开始重新记录和生成
reset master

# 删除指定日志 / 时间前的所有日志
purge master logs to 'filename'
purge master logs before 'yyyy-mm-dd hh:mi:ss'

# 清理指定文件之前的所有 binlog 文件 
purge master logs to "mysql-bin.000006"

# 清理指定时间前的所有 binlog 文件记录 
purge master logs before "2023-03-29 07:36:40"
```
#### 定点还原数据库

首先，清空数据库导入上一次备份，然后执行：
```sh
mysqlbinlog --stop-datetime="2023-07-02 15:27:48" /tmp/mysql-bin.000008 | mysql -u user -p password
```
数据库成功回滚到 2023-07-02 15:27:48 时刻

也可以先使用命令转换成 sql 文件
```sh
mysqlbinlog --start-datetime="2023-03-20 10:00:00" --stop-datetime="2023-03-21 10:00:00" mysql-bin.000001 -d database_name> filename_binlog.sql
```
database_name 为 binlog 文件中需要恢复数据的 mysql 数据库名称（可能存在多个库操作保存在一个 binlog 文件）

如果执行命令报错`unknown variable 'default-character-set=utf8'`，因为 mysqlbinlog 这个工具无法识别 binlog 配置中的 `default-character-set=utf8` 这个指令，在命令中加入 `--no-defaults` 即可。
```sh
mysqlbinlog --no-defaults --start-datetime="2023-03-20 10:00:00" --stop-datetime="2023-03-21 10:00:00" mysql-bin.000001 -d database_name> filename_binlog.sql
```

转换成的 sql 文件不能以直接导入 mysql 数据库的方式执行，它本质上还是一个二进制的文本文件，可以通过 `source filename_binlog.sql` 执行文件，这种方式在遇到错误时会继续执行，能比较好的避免报错停止执行的问题。

