# 迁移Zabbix数据库到TokuDB

## 背景介绍
线上的Zabbix数据库有几个大表数据量疯狂增长，单表已经超过500G，而且在早期也没做成分区表，后期维护非常麻烦。比如，想删除过期的历史数据，在原先的模式下，`history`、`history_uint`等几个大表是用 (itemid, clock) 两个字段做的联合主键，只用 clock 字段检索效率非常差。

TokuDB 是一个高性能、支持事务处理的 MySQL 和 MariaDB 的存储引擎。TokuDB 的主要特点是高压缩比，高 INSERT 性能，支持大多数在线修改索引、添加字段，特别适合像 Zabbix 这种高 INSERT，少 UPDATE 的应用场景。

## 迁移准备
欲使用 TokuDB 引擎，服务层可以选择和 MariaDB ，也可以选择 Percona ，我以往使用 Percona 的较多，因此本次也选择使用 Percona 版本集成 TokuDB 引擎。

按照正常方式安装即可，配置文件中增加3行：

```properties
malloc-lib= /usr/local/mysql/lib/mysql/libjemalloc.so
plugin-dir = /usr/local/mysql/lib/mysql/plugin/
plugin-load=ha_tokudb.so
```

如果不加载jemalloc，启动时就会有类似下面的报错：

```log
[ERROR] TokuDB not initialized because jemalloc is not loaded
[ERROR] Plugin 'TokuDB' init function returned error.
[ERROR] Plugin 'TokuDB' registration as a STORAGE ENGINE failed.
```

并且，修改内核配置，禁用transparent_hugepage，不关闭的话可能会导致TokuDB内存泄露（建议写到 /etc/rc.local 中，重启后仍可生效）：

```bash
echo never > /sys/kernel/mm/redhat_transparent_hugepage/defrag
echo never > /sys/kernel/mm/redhat_transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
```

如果不修改内核设置，启动时就会有类似下面的报错：

```log
Transparent huge pages are enabled, according to /sys/kernel/mm/redhat_transparent_hugepage/enabled
Transparent huge pages are enabled, according to /sys/kernel/mm/transparent_hugepage/enabled
[ERROR] TokuDB will not run with transparent huge pages enabled.
[ERROR] Please disable them to continue.
[ERROR] (echo never > /sys/kernel/mm/transparent_hugepage/enabled)
[ERROR]
[ERROR] ************************************************************
[ERROR] Plugin 'TokuDB' init function returned error.
[ERROR] Plugin 'TokuDB' registration as a STORAGE ENGINE failed.
```

然后，初始化数据库，启动即可。

我的服务器配置：E5-2620 * 2，64G内存，1T可用磁盘空间（建议datadir所在分区设置为xfs文件系统），下面是我使用的相关选项，仅供参考：

```
#
#my.cnf
# 
# Percona-5.6.17, TokuDB-7.1.6，用于Zabbix数据库参考配置
# 我的服务器配置：E5-2620 * 2，64G内存，1T可用磁盘空间（建议datadir所在分区设置为xfs文件系统）
# TokuDB版本：Percona-5.6.17, TokuDB-7.1.6(插件加载模式)
# 
# created by yejr(http://imysql.com), 2014/06/24
# 
[client]
port            = 3306
socket          = mysql.sock
#default-character-set=utf8
 
[mysql]
prompt="\\u@\\h \\D \\R:\\m:\\s [\\d]>
#pager="less -i -n -S"
tee=/home/mysql/query.log
no-auto-rehash
 
[mysqld]
open_files_limit = 8192
max_connect_errors = 100000
 
#buffer & cache
table_open_cache = 2048
table_definition_cache = 2048
max_heap_table_size = 96M
sort_buffer_size = 2M
join_buffer_size = 2M
tmp_table_size = 96M
key_buffer_size = 8M
read_buffer_size = 2M
read_rnd_buffer_size = 16M
bulk_insert_buffer_size = 32M
 
#innodb
#只有部分小表保留InnoDB引擎，因此InnoDB Buffer Pool设置为1G基本上够了
innodb_buffer_pool_size = 1G
innodb_buffer_pool_instances = 1
innodb_data_file_path = ibdata1:1G:autoextend
innodb_flush_log_at_trx_commit = 1
innodb_log_buffer_size = 64M
innodb_log_file_size = 256M
innodb_log_files_in_group = 2
innodb_file_per_table = 1
innodb_status_file = 1
transaction_isolation = READ-COMMITTED
innodb_flush_method = O_DIRECT

#tokudb
malloc-lib= /usr/local/mysql/lib/mysql/libjemalloc.so
plugin-dir = /usr/local/mysql/lib/mysql/plugin/
plugin-load=ha_tokudb.so
 
#把TokuDB datadir以及logdir和MySQL的datadir分开，美观点，也可以不分开，注释掉本行以及下面2行即可
tokudb-data-dir = /data/mysql/zabbix_3306/tokudbData
tokudb-log-dir = /data/mysql/zabbix_3306/tokudbLog
 
#TokuDB的行模式，建议用 FAST 就足够了，如果磁盘空间很紧张，建议用 SMALL
#tokudb_row_format = tokudb_small
tokudb_row_format = tokudb_fast
tokudb_cache_size = 44G
 
#其他大部分配置其实可以不用修改的，只需要几个关键配置即可
tokudb_commit_sync = 0
tokudb_directio = 1
tokudb_read_block_size = 128K
tokudb_read_buf_size = 128K

```

## 迁移过程
建议在一台全新的服务器上启动Percona(TokuDB)实例进程，初始化新的Zabbix数据库，直接将大表转成TokuDB引擎，并且开启分区模式。这样相比直接在线ALTER TABLE或者INSERT…SELECT导入数据都要来的快一些（我简单测试了下，差不多能快2-3倍，甚至更高）。

在做数据迁移时，建议在目标服务器上做库表结构初始化，在源服务器上采用分段方式导出，一个表导出多个备份文件，方便在恢复时可以并发导入。在导入时，并且记得临时关闭 binlog，最起码设置 sync_binlog = 0 以及 tokudb_commit_sync = 0，以提高导入速度。采用 mysqldump 增加 -w 参数即可实现根据条件分段导出，，或者是用MySQLDumper。

需要用到外键的表继续保留InnoDB引擎，其他表都可以转成TokuDB，history_str、trends、trends_uint、history、history_uint等几个大表是一定要转成TokuDB的，events由于需要用到外键，所以继续保留InnoDB引擎。

我将表结构初始化SQL脚本提供下载了，一份是没有采用分区表的，一份是采用分区表的，大家可自行选择。一般如果记录数超过1亿，就建议使用分区表，根据时间字段(clock)分区，方便后期维护，例如删除过期历史数据什么的。

## 收尾
然后就是观察下运行状态，是否还有个别慢查询堵塞。在我的环境中，一开始把items表也转成TokuDB了，结果有个画图的SQL执行计划不准确，非常慢。后来发现items表也需要用到外键，于是又转回InnoDB表，这个SQL也恢复正常了。

