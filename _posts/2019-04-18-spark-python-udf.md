---
comments: true
layout: post
title: Note about Spark Python UDF 
--- 

The other day, my colleague was developing a PySpark(2.3.1) application which reads Chinese sentences from a Hive table, tokenizes them with a Python UDF and saves the first words into another table. The codes are roughly like this.

```python
def tokenize(sentence):
    return tokenizer.tokenize(sentence)[0]

df = spark.sql("select sentence from db.article")
df = df.repartition(1000)

tokenize_udf = udf(tokenize, StringType())
df = df.withColumn('word', tokenize_udf('sentence'))
df.filter(df.word != '')

df.write.format('orc').mode('overwrite').saveAsTable('db.words')        
```

The Python UDF was time-consuming so my colleague tried increasing parallelism with `repartition` from 99 to 1000. It didn't work as expected and the job still stuck in those 99 tasks. 

I looked into the physical plan and it turned out that the Python UDF (`BatchEvalPython`) had been executed twice, once in a push down filter before repartition and once after repartition. 

```
== Physical Plan ==
Execute CreateDataSourceTableAsSelectCommand CreateDataSourceTableAsSelectCommand `db`.`words`, Overwrite, [word#19]
+- *(3) Project [sentence#13, pythonUDF0#26 AS word#19]
   +- BatchEvalPython [tokenize(sentence#13)], [sentence#13, pythonUDF0#26]
      +- Exchange RoundRobinPartitioning(1000)
         +- *(2) Project [sentence#13]
            +- *(2) Filter NOT (pythonUDF0#25 = )
               +- BatchEvalPython [tokenize(sentence#13)], [sentence#13, pythonUDF0#25]
                  +- *(1) Filter (isnotnull(sentence#13) && NOT (sentence#13 = ))
                     +- HiveTableScan [sentence#13], HiveTableRelation `db`.`sentence`, org.apache.hadoop.hive.ql.io.orc.OrcSerde, [sentence#13]
```

I found [another issue when Python UDF is used in filter](https://issues.apache.org/jira/browse/SPARK-22541) which led me to [an important note about Python UDF](https://github.com/apache/spark/blob/v2.3.1/python/pyspark/sql/functions.py#L2113)

```python
def udf(f=None, returnType=StringType()):
    """Creates a user defined function (UDF).
    .. note:: The user-defined functions are considered deterministic by default. Due to
        optimization, duplicate invocations may be eliminated or the function may even be invoked
        more times than it is present in the query. If your function is not deterministic, call
        `asNondeterministic` on the user defined function. E.g.:
```

Hence, it's possible **Python UDF is invoked more times than it is present in the query**. There are more notes you should check out when your Spark application with Python UDF behaves strangely next time.

### Solution

To increase parallelism of executing Python UDF, we can decrease the input split size as follows.

```
spark.sparkContext._jsc.hadoopConfiguration().set("mapred.max.split.size", "33554432")
```

Thanks to this StackOverflow answer on [How to set Hadoop configuration values from PySpark](https://stackoverflow.com/questions/28844631/how-to-set-hadoop-configuration-values-from-pyspark).