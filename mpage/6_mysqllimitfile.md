有时候，我们会遇到类似下面的报错信息：

```
.....
[ERROR] /usr/local/mysql/bin/mysqld: Can't open file: './yejr/access.frm' (errno: 24)
[ERROR] /usr/local/mysql/bin/mysqld: Can't open file: './yejr/accesslog.frm' (errno: 24)
......
[ERROR] Error in accept: Too many open files
....
```

提示信息很明显，打开文件数达到上限了，需要提高上限，或者释放部分已打开的表文件描述符。

在MySQL中，有几个地方会存在文件描述符限制：

1. 在Server层，整个mysqld实例打开文件总数超过用户进程级的文件数限制，需要检查内核 fs.file-max 限制、进程级限制 ulimit -n 及MySQL中的 open-files-limit 选项，是否有某一个超限了。任何一个条件超限了，就会抛出错误。

2. 虽然Server层总文件数没有超，但InnoDB层也有限制，所有InnoDB相关文件打开总数不能超过 innodb-open-files 选项限制。否则的话，会先把最早打开的InnoDB文件描述符关闭，才能打开新的文件，但不会抛出错误，只有告警信息。

相应地，如果提示超出限制，则可以使用下面方法提高上限：

1. 首先，提高内核级总的限制。执行：sysctl -w fs.file-max=3264018；

2. 其次，提高内核对用户进程级的打开文件数限制。执行：ulimit -n 204800；

3. 最后，适当提高MySQL层的几个参数：open-files-limit、innodb-open-files、table-open-cache、table-definition-cache。

关于前面两个限制网上可以找到很多详细解释，我就不多说了，重点来说下MySQL相关的4个选项。

1. open-files-limit
它限制了mysqld进程可持有的最大打开文件数，相当于是一个小区的总电闸，一旦超限，小区里所有住户都得停电。
5.6.7（含）以前，默认值0，最大和OS内核限制有关；
5.6.8（含）以后，默认值会自动计算，最大和OS内核限制有关。

在5.6.8及以后，其自动计算的几个限制规则见下，哪个计算结果最大就以哪个为上限：

* 10 + max_connections + (table_open_cache * 2)
* max_connections * 5
* open_files_limit value specified at startup, 5000 if none
 

2. innodb-open-files

	限制InnoDB引擎中表空间文件最大打开的数量，相当于自己家中电箱里的某个电路保险，该电路短路的话，会自动跳闸，而不会影响其他电路，去掉短路源后重新按上去就可以使用。

	其值最低20，默认400，只计算了包含ibdata*、ib_logfile*、*.ibd 等三类文件，redo log不计算在内，5.6以后可独立undo log，我还未进行测试，应该也不会被计算在内，有兴趣的朋友可验证下。

3. table-definition-cache

	该cache用于缓存 .frm 文件，该选项也预示着 .frm 文件同时可打开最大数量。
	* 5.6.7 以前默认值400；
	* 5.6.7 之后是自动计算的，且最低为400，自动计算公式：400 + (table-open-cache / 2)。

	对InnoDB而言，该选项只是软性限制，如果超过限制了，则会根据LRU原则，把旧的条目删除，加入新的条目。

	此外，innodb-open-files 也控制着最大可打开的表数量，和 table-definition-cache 都起到限制作用，以其中较大的为准。如果没配置限制，则通常选择 table-definition-cache 作为上限，因为它的默认值是 200，比较大。

4. table-open-cache
   
	该cache用于缓存各种所有数据表文件描述符。
	* 5.6.7 以前，默认值400，范围：1 – 524288；
	* 5.6.8 – 5.6.11，默认值2000，范围：1 – 524288；
	* 5.6.12以后，默认值2000（且能自动计算），范围：1 – 524288。

补充说明1：关于如何计算表文件描述符的建议：

table-open-cache 通常和 max-connections 有关系，建议设置为 max_connections * N，N的值为平均每个查询中可能总共会用到的表数量，同时也要兼顾可能会产生临时表。

补充说明2：MySQL会在下列几种情况把表从table cache中删掉：

1. table cache已满，并且正要打开一个新表时；
2. table cache中的条目数超过 table_open_cache 设定值，并且有某些表已经长时间未访问了；

3. 执行刷新表操作时，例如执行 FLUSH TABLES，或者 mysqladmin flush-tables 或 mysqladmin refresh
 

补充说明3：MySQL采用下述方法来分配table cache：

1. 当前没在用的表会被释放掉，从最近最少使用的表开始；

2. 当要打开一个新表，当前的cache也满了且无法释放任何一个表时，table cache会临时加大，临时加大的table cache中的表不用了之后，会被立刻释放掉。