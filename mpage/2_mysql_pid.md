# MySQL进程号、连接ID、查询ID、InnoDB线程与系统线程如何对应

> 一文快速掌握 MySQL进程号、连接ID、查询ID、InnoDB线程与系统线程的对应关系。

有时候，怀疑某个MySQL内存查询导致CPU或磁盘I/O消耗特别高，但又不确定具体是哪个SQL引起的。

或者当InnoDB引擎内部有semaphore wait时，想知道具体是哪个线程/查询引起的。多说一下，当有semaphore wait事件超过600秒的话，InnoDB会发出crash信号：

```ts
InnoDB: ###### Diagnostic info printed to the standard error stream
2020-12-13T09:41:33.810011Z 0 [ERROR] [FATAL] InnoDB: Semaphore wait has lasted > 600 seconds. We intentionally crash the server because it appears to be hung.
2020-12-13 10:41:33 0x7f3d92a4e700 InnoDB: Assertion failure in thread 139902430013184 in file ut0ut.cc line 917
InnoDB: We intentionally generate a memory trap.
InnoDB: Submit a detailed bug report to http://bugs.mysql.com.
InnoDB: If you get repeated assertion failures or crashes, even
InnoDB: immediately after the mysqld startup, there may be
InnoDB: corruption in the InnoDB tablespace. Please refer to
InnoDB: http://dev.mysql.com/doc/refman/8.0/en/forcing-innodb-recovery.html
InnoDB: about forcing recovery.
09:41:33 UTC - mysqld got signal 6 ;
```
因此也要监控InnoDB的`semaphore wait`状态，一旦超过阈值，就要尽快报警并分析出问题原因，及时杀掉或停止引起等待的查询请求。

不过本文想讨论的是，MySQL的进程ID、内部查询ID、内部线程ID，和操作系统层的进程ID、线程如何对应起来。

MySQL是一个单进程多线程的服务程序，用 `ps -ef|grep mysqld` 就能看到其系统进程ID了。另外，当 `my.cnf` 配置文件中增加一行 `innodb_status_file = 1` 时，也会生成带有系统进程ID的innodb status 文件

```bash
[root@yejr.run]# ps -ef | grep mysqld
mysql    38801     1  0 Jun13 ?        00:03:30 /usr/local/GreatSQL-8.0.22/bin/mysqld --defaults-file=/mysql/data06/my.cnf

[root@yejr.run]# ls -la innodb_status.38801
-rw-r----- 1 mysql mysql 4906 Jun 14 14:26 innodb_status.38801
```

文件 `innodb_status.pid` 的作用是每隔15秒左右输出innodb引擎各种状态信息，和执行 `SHOW ENGINE INNODB STATUS` 的作用相同。二者的区别在于，前者（文件输出方式）的输出内容长度不受限制，而后者（命令行输出）则最多只显示1MB内容，更多的会被截断。所以务必设置 `innodb_status_file = 1` 选项。

```ts
Standard Monitor output is limited to 1MB when produced using the SHOW ENGINE INNODB STATUS statement. This limit does not apply to output written to server standard error output (stderr).
```