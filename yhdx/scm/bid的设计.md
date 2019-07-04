1、需求

订单的id一般是数据库的主键，是一个系统级别的id，不便于人类识别。增加一个按天重置累计的可识别的id，用于人类可识别的id，例如SO20190701000123。



2、实现

基于redis的incr命令，因为redis是单线程的，且incr命令也是一个原子操作；用日期作为redis键，存储每日当前的累计数。因为redis是基于内存存储的（当然也可以持久化到磁盘），在数据库也存储备份了一份（异步的写入）。代码片段如下：

```java
//基于redis计数
long sequence = bidManager.updateForIncr(key, now);
//异步把最新序列号写入MySQL数据库
pool.execute(new DataSyncWorker(key, sequence));
```

因为redis的写和MySQL的写并没有事务保证，因此redis的最新计数并不能保证正确写入数据库。如果redis发生了故障或重启，当redis数据恢复时，需要从MySQL读取，一般是在MySQL读取到的值上增加一个偏移量，比如100，防止序列号的重复使用；然后写入redis，作为redis中的计数起始值。