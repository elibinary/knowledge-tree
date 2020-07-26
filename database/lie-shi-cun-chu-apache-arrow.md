# 列式存储 & Apache Arrow

参数时听到 PingCAP 的 topic 讲到 TiDB 使用 [Apache Arrow](https://arrow.apache.org/) 结构格式来存储数据，非常感兴趣，会后看了一些简介和文档

> Apache Arrow: A cross-language development platform for in-memory data. It specifies a standardized language-independent columnar memory format for flat and hierarchical data, organized for efficient analytic operations on modern hardware.

Apache Arrow 是一种基于内存的列式数据存储结构，再继续之前还是先来看下列式存储与行式存储

> OLTP: On-Line Transaction Processing OLAP: On-Line Analytical Processing

### 行式存储

传统 OLTP 数据库通常使用的是 `行式存储` 这种存储方式把所有 column 排列组成一行（一条）数据让后进行存储，并配合 B+ Tree （如 MySQL）或者 SS-Tree 索引来实现快速查询

举个例子，现在我们有一份用户数据存储在行式存储数据库中

```text
+----+----------+------------+
| id | nickname | avatar_url |
+----+----------+------------+
|  1 | b2b406   | img1.png    |
|  2 | f537df   | img2.png    |
|  3 | c59db7   | img3.png    |
|  4 | 55c184   | img4.png    |
|  5 | a6920f   | img5.png    |
+----+----------+------------+
```

那么它将以每一行的值串联的形式存储进内存或磁盘，然后存储下一行

```text
1,b2b406,img1.png;2,f537df,img2.png...
```

行式存储对于绝大多数 OLTP 场景都非常实用，性能良好。因为这种场景大多是基于处理单一实体的多个属性，这种应用程序某一实体（基于行）工作做，所以在从磁盘获取数据时操作的存储页很少很高效

### 列式存储

> 列式数据库可以是关系型、也可以是 NoSQL，这和是否是列式并无关系

与行式存储不同的是，列式存储会把一列中的数据串起来存储，然后再存储下一列，还是以上面那份数据为例

```text
+----+----------+------------+
| id | nickname | avatar_url |
+----+----------+------------+
|  1 | b2b406   | img1.png    |
|  2 | f537df   | img2.png    |
|  3 | c59db7   | img3.png    |
|  4 | 55c184   | img4.png    |
|  5 | a6920f   | img5.png    |
+----+----------+------------+
```

按列存储

```text
1,2,3,4,5;b2b406,f537df,c59db7,55c184,a6920f;...
```

这种存储方式非常适合 OLAP 类场景，此类场景常常需要对某一列或某几列数据进行分析处理

如果使用行式存储那么在处理时将不得不把每一行的数据都整个取出来，I/O 利用率很低，并且即使数据都在内存中，也需要消耗大量 CUP 资源来将一行中的所有列拼接起来

当然基于行的数据库在对列做某一些操作时并不一定会真的把整行数据扫出来操作，它们增加了很多方法来处理常用列式操作，比如 `sum`

另一方面来说，单纯给列式存储的表增加索引存储，并不能使 OLTP 类操作很高效，行信息都分散在很多存储页中，并且取出来后也同样需要消耗 CPU 资源把数据拼起来

列式存储的优点：

1. 拥有极高的装载速度
2. 同时因为数据同格式聚合，可以实现极高的压缩率，不仅节省储存空间也节省计算内存和CPU
3. 非常适合做聚合操作

但是它同样在某些方面表现不佳：

1. 不适合做小量数据扫描
2. 实时更新及删除的效率很差

### Apache Arrow

> Columnar memory layout allows applications to avoid unnecessary IO and accelerate analytical processing performance on modern CPUs and GPUs.

!\[[Performance Advantage of Columnar In-Memory](https://static001.infoq.cn/resource/image/c6/57/c6c772ff7ca8338b94bc9900ffbf3c57.png)\]\[2\]

这种来自 Arrow 官网的图很清晰的呈现出两种存储方式的差异

Apache Arrow 不是一个存储系统，而是处理分层的列式内存数据的一系列格式和算法。

下面就暂时先不对 Apache Arrow 的具体结构及算法特性展开展开了，另外现在已经有许多开源项目都已经支持了 Arrow，有兴趣可以深入了解一下

---

* BigTable（HBase）是列式存储吗？

