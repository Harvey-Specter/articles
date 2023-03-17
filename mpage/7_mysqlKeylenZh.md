# 解读EXPLAIN执行计划中的key_len
> EXPLAIN中的key_len一列表示什么意思，该如何解读？

EXPLAIN执行计划中有一列 key_len 用于表示本次查询中，所选择的索引长度有多少字节，通常我们可借此判断联合索引有多少列被选择了。

在这里 key_len 大小的计算规则是：

* 一般地，key_len 等于索引列类型字节长度，例如int类型为4-bytes，bigint为8-bytes；
* 如果是字符串类型，还需要同时考虑字符集因素，例如：CHAR(30) UTF8则key_len至少是90-bytes；
* 若该列类型定义时允许NULL，其key_len还需要再加 1-bytes；
* 若该列类型为变长类型，例如 VARCHAR（TEXT\BLOB不允许整列创建索引，如果创建部分索引，也被视为动态列类型），其key_len还需要再加 2-bytes;

|  列类型   | key_len  | 备注| 
|---| ----  |-----|
| id int  | key_len = 4+1 = 5 |允许NULL，加1-byte|
| id int not null  | key_len = 4 |不允许NULL|
| user char(30) utf8  | key_len = 30*3+1 |允许NULL|
| user varchar(30) not null utf8  | key_len = 30*3+2 |动态列类型，加2-bytes|
| user varchar(30) utf8  | key_len = 30*3+2+1 |动态列类型，加2-bytes；允许NULL，再加1-byte|
| detail text(10) utf8  | key_len = 30*3+2+1 |TEXT列截取部分，被视为动态列类型，加2-bytes；且允许NULL|


**备注，key_len 只指示了WHERE中用于条件过滤时被选中的索引列，是不包含 ORDER BY/GROUP BY 这部分被选中的索引列。
例如，有个联合索引 idx1(c1, c2, c3)，3个列均是INT NOT NULL，那么下面的这个SQL执行计划中，key_len的值是8而不是12：**

> SELECT…WHERE c1=? AND c2=? ORDER BY c1;