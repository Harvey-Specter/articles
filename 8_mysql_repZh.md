# 如何保证主从复制数据一致性
> MySQL主从复制环境中，如何才能保证主从数据的一致性呢？

## 关于主从复制
现在常用的MySQL高可用方案，十有八九是基于 MySQL的主从复制（replication）来设计的，包括常规的一主一从、双主模式，或者半同步复制（semi-sync replication）。

我们常常把MySQL replication说成是MySQL同步（sync），但事实上这个过程是异步（async）的。大概过程是这样的：

1. 在master上提交事务后，并且写入binlog，返回事务成功标记；
2. 将binlog发送到slave，转储成relay log；
3. 在slave上再将relay log读取出来应用。

步骤1和步骤3之间是异步进行的，无需等待确认各自的状态，所以说MySQL replication是异步的。

MySQL semi-sync replication在之前的基础上做了加强完善，整个流程变成了下面这样：

1. 首先，master和至少一个slave都要启用semi-sync replication模式；
2. 某个slave连接到master时，会主动告知当前自己是否处于semi-sync模式；
3. 在master上提交事务后，写入binlog后，还需要通知至少一个slave收到该事务，等待写入relay log并成功刷新到磁盘后，向master发送“slave节点已完成该事务”确认通知；
4. master收到上述通知后，才可以真正完成该事务提交，返回事务成功标记；
5. 在上述步骤中，当slave向master发送通知时间超过rpl_semi_sync_master_timeout设定值时，主从关系会从semi-sync模式自动调整成为传统的异步复制模式。

半同步复制看起来很美好有木有，但如果网络质量不高，是不是出现抖动，触发上述第5条的情况，会从半同步复制降级为普通复制；此外，采用半同步复制，会导致master上的tps性能下降非常严重，最严重的情况下可能会损失50%以上。

这样来看，除非需要非常严格保证数据一致性等迫不得已的场景，就不太建议使用半同步复制了。当然了，事实上我们也可以通过加强程序端的逻辑控制，来避免主从数据不一致时发生逻辑错误，比如说如果在从上读取到的数据和主不一致的话，那么就触发主从间的一次数据修复工作。或者，我们也可以用 pt-table-checksum & pt-table-sync 两个工具来校验并修复数据，只要运行频率适当，是可行的。

真想要提高多节点间的数据一致性，可以考虑采用PXC方案。现在已知用PXC规模较大的有qunar、sohu，如果团队里初期没有人能比较专注PXC的话，还是要谨慎些，毕竟和传统的主从复制差异很大，出现问题时需要花费更多精力去排查解决。

## 如何保证主从复制数据一致性
上面说完了异步复制、半同步复制、PXC，我们回到主题：在常规的主从复制场景里，如何能保证主从数据的一致性，不要出现数据丢失等问题呢？

在MySQL中，一次事务提交后，需要写undo、写redo、写binlog，写数据文件等等。在这个过程中，可能在某个步骤发生crash，就有可能导致主从数据的不一致。为了避免这种情况，我们需要调整主从上面相关选项配置，确保即便发生crash了，也不能发生主从复制的数据丢失。

### 1. 在master上修改配置  

        innodb_flush_log_at_trx_commit = 1
        sync_binlog = 1  

上述两个选项的作用是：保证每次事务提交后，都能实时刷新到磁盘中，尤其是确保每次事务对应的binlog都能及时刷新到磁盘中，只要有了binlog，InnoDB就有办法做数据恢复，不至于导致主从复制的数据丢失。

### 2. 在slave上修改配置

        master_info_repository = "TABLE"
        relay_log_info_repository = "TABLE"
        relay_log_recovery = 1

上述前两个选项的作用是：**确保在slave上和复制相关的元数据表也采用InnoDB引擎，受到InnoDB事务安全的保护**，而后一个选项的作用是**开启relay log自动修复机制，发生crash时，会自动判断哪些relay log需要重新从master上抓取回来再次应用，以此避免部分数据丢失的可能性。**

通过上面几个选项的调整，就可以确保主从复制数据不会发生丢失了。但是，**这并不能保证主从数据的绝对一致性**，因为，有可能设置了ignore\do\rewrite等replication规则，或者某些SQL本身存在不确定因素，或者人为在slave上修改数据，最终导致主从数据不一致。这种情况下，可以采用pt-table-checksum 和 pt-table-sync 工具来进行数据的校验和修复。