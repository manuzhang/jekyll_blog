---
comments: true
layout: post
title: Benchmarking PostgreSQL & Cassandra (Part Two)
---

下面讲讲 Cassandra 这部分我是怎么做的。（单个结点）

## I. 建表

TPC-H 提供的是关系数据库中的表，需要修改成 Cassandra 的 column family。

我把关系数据库中的 primary key 对应成 Cassandra 中的 row key，其余的 column 保持不变。 对于多个属性做 primary key 的情形，我将它们用 “：” 连接做 row key。由于 CLI 支持批处理，我把建表的语句写成脚本，一次执行。
` bin/cassandra-cli -h $HOST -p $PORT --batch < $FILE `
`$HOST`， `$PORT`， `$FILE` 用具体的 IP 地址，端口号和脚本文件代替。

Cassandra 也支持用 CQL 建表，语法和关系数据库中类似，具体参见 [The evolution of schema in Cassandra](http://www.datastax.com/dev/blog/schema-in-cassandra-1-1)。

## II. 导入数据

不需要再生成数据，拿前面给 postgreSQL 生成的即可。

Cassandra 提供了 Bulk load 的接口 SSTableWriter，直接将数据写入到 SSTable。具体步骤在 [Using the Cassandra Bulk Loader](http://www.datastax.com/dev/blog/bulk-loading)。

思路很简单，就是一行一行的解析数据，把 row key 和 column{name, value, timestamp} 提交给 SSTableWriter，由它统一写入 Cassandra。就是要给每张表写个解析方法有点累。

## III. 连接运算


前面都是准备工作，终于到了核心步骤。我是在客户端调用 Cassandra Thrift API 做。

传统关系数据库都有 query processor，会生成一个最优的查询计划。Cassandra 没有，只好由人脑完成，这部分时间就不计了。

需要处理的是类似这样一条语句：

```sql
SELECT * FROM 
(SELECT * FROM customer LIMIT %d) tmp1, 
(SELECT * FROM orders LIMIT %d) tmp2 
WHERE c_custkey = o_custkey 
```

原来 customer 这张表有 150000 行，8 个属性，c_custkey 是 primary key。orders 这张表有 1500000 行，9 个属性，o_orderkey 是 primary key，o_custkey 是 foreign key。实验的时候，我在两张表都选取了 150000 行。

由于选取了所有属性，我首先需要读取两个 column family，用的是 get_range_slice 方法，比较郁闷的是它不用 row cache。

连接开始用的嵌套连接，从没有算出结果。果断改成散列连接。具体来说，就是

1. 扫描一遍 customer， 建一个 HashMap，键是 row key (c_custkey)，值是 row；
2. 扫描一遍 orders，取出每个 row 中 o_custkey column 的值；
3. 如果在 HashMap 命中的话，就取出 HashMap 中的 row 和这个 row，都加到结果集里面。

## IV. 结果分析

实验结果：总时间在 10s 到 15s，读一个 column family 在 6s 到 9s，散列连接 0.1s 到 0.2s。postgreSQL 的表现在 2s 左右。
很明显，读操作是瓶颈。既然 Cassandra 和 postgreSQL 都是在读磁盘，为什么性能会有差呢？

我在 [stackoverflow](http://stackoverflow.com/questions/12616699/how-cassandra-deals-with-disk-io/12646055#comment17358380_12646055) 上问了这个问题，由于我拿 Cassandra 做 join，还是用一个结点做的，引起了不少误会（我一点都没有 Cassandra 慢的意思，只是想知道两者读磁盘机制的差别）。个人觉得比较合理的解释是，Cassandra 的连接在客户端，两个 column family 读到 JVM Heap 里后，会使其使用量增加 700 MB 左右， JVM 会用不少时间做 garbage collection。postgreSQL 读到客户端的就已经是连接结果了，而连接那部分操作和 JVM 没有关系。

## V. 反思

Cassandra 本质上是一个 Key-Value Store，是一个 Distributed Hashtable，就读而言，它最擅长的是在某个 key 上的 get 操作。

我们为什么需要一个 Key-Value Store？那天在微博上看到NoSQL发端大概是 [Scalable, distributed data structures for internet service construction](http://dl.acm.org/citation.cfm?id=1251251) 这篇论文。

其中比较了关系数据库，文件系统和论文提出的分布式数据结构。关系数据库给用户提供了很好用的接口，数据的独立性高；文件系统的接口很底层，用户需要自己组织数据；分布式数据结构介于两者之间。 论文针对的应用环境是 Internet Service，scalability 是我所理解的核心问题，当请求数量成倍增加时，性能下降是平稳的。

再回到 Cassandra 上，实际中，一个用户请求需要的很可能只是一个或几个 column 的数据，几乎没有要扫描整个 column family 的，Cassandra 要保证的是即使在请求数量激增的时候，单个用户也不会明显感受到性能的下降。


## VI. 未来

不知道。Redis？也许吧。postgreSQL 最近有了开源的集群解决方案 [postgres-xc](http://postgres-xc.sourceforge.net/)（商业版有 greenplum)，可以看看。Eddies 的源码也一直都没有读。


