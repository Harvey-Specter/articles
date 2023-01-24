# 监控平均SQL响应时长

> MySQL里如何监控平均SQL响应时长？

## 问题由来
对MySQL的性能指标监控，除了关注tps（每秒可执行的事务数）、qps（每秒请求数）两个衡量吞吐量的重要指标外，还应该监控平均SQL响应时长。

# 怎么做
有几个可选方案：

1. 利用MySQL提供的benchmark()函数。这个函数的作用是模拟进行N次某种调用，这样一来，我们就可以利用这个函数调用N次专门的存储过程，根据其执行耗时，以此作为平均SQL响应时长的依据；

2. 利用pt-query-digest工具，并结合tcpdump实时抓取每个SQL请求，也就能分析出每个SQL请求的响应时长了；

3. 使用Percona或者MariaDB分支版本提供的QUERY_RESPONSE_TIME插件功能，它可以帮我们统计平均SQL响应时长的分布区间，类似直方图功能；

第一种相对比较简单但不够精确（不过也是够用的），第二种略麻烦些但可以看到每次请求的详细记录，第三种则只能看到整体的分布，无法看到每次请求的详细记录。

写在最后

监控性能指标时，除了关注吞吐量，还应该关注每次请求的平均响应时长。以高速公路收费站为例，有几个收费口基本可表示其并发收费能力（tps），而每辆车的平均通行时间如果很久的话，相信你也是受不了的 ：）