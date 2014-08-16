---
comments: true
layout: post
title: Benchmarking PostgreSQL & Cassandra (Part One)
---

前面一直在读 Cassandra 的源码，了解了其架构和实现 （相关的日志大多还停留在草稿阶段，希望能尽快整理出来），下面结合前面的 Eddies 知识探索在 Cassandra 上完成适应性查询（连接和聚合）。

第一步是简单的测评一下 Cassandra 的连接速度，为了能和传统关系型数据库有个比较，采用 [TPC-H](http://www.tpc.org/tpch/) 对 PostgreSQL 和 Cassandra 做 [Benchmarking](http://en.wikipedia.org/wiki/Benchmarking)。

TPC-H 主要是对商业决策进行测评，原生支持 Oracle， DB2， SQLServer 这些商业数据库，没有 PostgreSQL 和 MySQL 这样的开源数据库，更不用说 Cassandra 等 NoSQL DBMS （没有连接聚合，不支持 ACID)。当然，用 TPC-H 给 Cassandra 做 Benchmarking 是否合适也值得讨论。[YCSB](https://github.com/brianfrankcooper/YCSB/wiki) 提供了对 "cloud data serving" 系统 （Cassandra，HBase，PNUTS，sharded MySQL）Benchmarking，但也仅限于简单的 CRUD 操作，没有相对复杂的连接聚合运算。其实，如前所述，在 Cassandra 上做连接和聚合本身就是探索性质的，在 TPC-H 上试试也无妨。

下面先讲讲 PostgreSQL 部分，参考在[这里](http://dsl.serc.iisc.ernet.in/projects/PICASSO/picasso_download/doc/Installation/tpch.htm).

## I. 下载

[下载](http://www.tpc.org/tpch/spec/tpch_2_14_3.tgz) TPC-H 的数据生成工具 DBGEN 和查询语句生成工具 QGEN。解压后生成两个文件夹 `dbgen` 和 `ref_data`，两个工具都在 `dbgen` 中，先打开 README ， 其中有工具的介绍和使用说明。
`ref_data` 下是源数据：

```bash
ls ref_data/
1  100  1000  10000  100000  300  3000  30000
```

从`1` 的数据生成的数据量是 1GB 左右，`1000`就是 1TB。

## II. 创建表格

所有创建表格的语句都在 `dbgen` 下的 `dss.ddl` 文件中，在 PostgreSQL 中可以直接执行 (Ubuntu 下 PostgreSQL 的安装参考[官方文档](http://wiki.ubuntu.org.cn/PostgreSQL)，这里不再赘述了）。

```
postgres=# \i /home/manuzhang/tpch/dbgen/dss.ddl 
```

要查看刚创建的表格，可以用

```sql
select schemaname, relname
from pg_stat_user_tables
```

或者更通用一点的（会包含系统表）

```sql
select  table_schema, table_name
from information_schema.tables
```

这里还没有加约束性条件，可以加快后面加载数据的速度。

## III. 生成数据

先编译 DBGEN 和 QGEN，`dbgen` 下有一个模板 `makefile.suite`，需要添加三个参数：

```
CC      = gcc
# Current values for DATABASE are: INFORMIX, DB2, TDAT (Teradata)
#                                  SQLSERVER, SYBASE, ORACLE, VECTORWISE
# Current values for MACHINE are:  ATT, DOS, HP, IBM, ICL, MVS, 
#                                  SGI, SUN, U2200, VMS, LINUX, WIN32 
# Current values for WORKLOAD are:  TPCH
DATABASE= INFORMIX
MACHINE = LINUX
WORKLOAD = TPCH
```

这里的 `DATABASE` 选项没有 PostgreSQL，试着随便选一个，后面也没有出现问题。
下面用 DBGEN 生成数据：

```
./dbgen
```

可以看到生成了8个 `.tbl` 文件，它们就是 TPC-H 用到的8张表，文件中的行与表格的行相对应，属性间用 '|' 隔开。

## IV. 导入数据

导入数据可以使用 PostgreSQL 的 COPY 命令：

```
COPY '' FROM '' WITH DELIMITER AS '';
```

这里还有一个问题，如 `nation.tbl`，在一行的结尾还有一个 '|'， PostgreSQL 会以为还有一列，这样就和开始定义的表结构冲突了。（感觉像是生成程序里没有加最后一个属性不加分隔符的判断，为什么只有 PostgreSQL 不能理解）

```
0|ALGERIA|0| haggle. carefully final deposits detect slyly agai|
1|ARGENTINA|1|al foxes promise slyly according to the regular accounts. bold requests alon|
2|BRAZIL|1|y alongside of the pending deposits. carefully special packages are about the ironic forges. slyly special |
3|CANADA|1|eas hang ironic, silent packages. slyly regular packages are furiously over the tithes. fluffily bold|
4|EGYPT|4|y above the carefully unusual theodolites. final dugouts are quickly across the furiously regular d|
...
```

开始想的是去修改表的结构，增加一列空值，这样太丑陋了，就用脚本去掉尾巴上的 '|'：

```bash
DIR=.
for FILE in $( find $DIR -name '*.tbl')
do
   mv $FILE $FILE.old
   sed -e 's/[|]$//' $FILE.old > $FILE
done
```

然后写一个 JDBC 程序加载数据。

## V. 添加约束

语句在 `dss.ri` 中，还不符合 PostgreSQL 语法，主要是 PostgreSQL 中不给外键命名 （语法参考 PostgreSQL 的文档）。修改之后像前面一样运行即可。

这样表就建好了，下面用 QGEN 生成查询。

## VI. 生成查询

在 `queries` 文件夹下是22条查询语句的模板，语句的结构已经写好了，其中的一些参数是 QGEN 随机生成的。不幸的是，它的语法也不是 PostgreSQL，需要修改。

直接运行 QGEN 或是在 `queries` 文件下运行 QGEN 都会报错，可行的是把 `queries` 下的文件复制到上一层文件夹再运行 QGEN。

这里是生成查询的脚本：


```bash
DIR=.
mkdir $DIR/finals
cp $DIR/queries/*.sql $DIR
for FILE in $(find $DIR -maxdepth 1 -name "[0-9]*.sql")
do
    DIGIT=$(echo $FILE | tr -cd '[[:digit:]]')
    ./qgen $DIGIT > $DIR/finals/$DIGIT.sql
done
rm *.sql
```


## VII. 还有什么

基本上都做好了，难道接下去就是执行查询语句，看一下运行时间吗？ 探索中。。。

另外，我做了个 patch，用来修改查询语句，和上面的脚本都放在 [GitHub](https://github.com/manuzhang/tpch4postgres) 上了。



