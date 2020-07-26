# MySQL master-slave lag

MySQL master-slave 同步的执行过程：

整个过程主要涉及到两个线程：

* IO\_THREAD: 从 master 读取 binlog file 并写入本地中继日志
* SQL\_THREAD: 从中继日志读取的事件，并在 slave 端执行

当发生 lag 时，第一步就需要先确认是哪个线程的问题

#### IO\_THREAD 延迟

主要可能是两个服务器之间的网络链接状况差，延迟高

#### SQL\_THREAD 延迟

可能性较多

1. 主从两个机子的硬件性能差别较大，导致同样的 binlog 事件在 slave 上需要更多的执行时间
2. 可能是由于不同的索引配置导致执行时间的差异
3. 也可能是负荷不同
4. 大事务也会导致 block SQL\_THREAD，从而影响同步速度

#### 原因查找

使用 pt stalk 收集 lag 发生时，mysql 相关的诊断数据

> pt-stalk - Collect forensic data about MySQL when problems occur.

[https://www.percona.com/doc/percona-toolkit/2.2/pt-stalk.html](https://www.percona.com/doc/percona-toolkit/2.2/pt-stalk.html)

参考： [https://www.percona.com/blog/2014/05/02/how-to-identify-and-cure-mysql-replication-slave-lag/](https://www.percona.com/blog/2014/05/02/how-to-identify-and-cure-mysql-replication-slave-lag/)

