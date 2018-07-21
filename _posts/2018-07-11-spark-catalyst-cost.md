---
comments: true
layout: post
title: The hidden cost of Spark withColumn
--- 

Recently, we've been working on machine learning pipeline with Spark, where [Spark SQL & DataFrame](https://spark.apache.org/sql/) is used for data preprocessing and [MLLib](https://spark.apache.org/mllib/) for training. In one use case, the data source is a very wide Hive table of ~1000 columns. The columns are stored in String so we need to cast them to Integer before they can be fed into model training.

This was what I got initially with DataFrame Scala API (2.2.0).

```scala
df.columns.foldLeft(df) { case (df, col) =>
  df.withColumn(col, df(col).cast(IntegerType))
}
```

Since DataFrame is immutable, I created a new `DataFrame` for each `Column` casted using `withColumn`. I didn't think that was be a big deal since at run time all columns would be casted in one shot thanks to [Spark's catalyst optimizer](https://databricks.com/blog/2015/04/13/deep-dive-into-spark-sqls-catalyst-optimizer.html). 

> At its core, Catalyst contains a general library for representing trees and applying rules to manipulate them. On top of this framework, we have built libraries specific to relational query processing (e.g., expressions, logical query plans), and several sets of rules that handle different phases of query execution: analysis, logical optimization, physical planning, and code generation to compile parts of queries to Java bytecode

To my surprise, the job stuck in submission for minutes without outputting anything. Luckily, I have a nice colleague Vincent who saved my day with the following fix.  

```scala
df.select(df.columns.map { col =>
  df(col).cast(IntegerType)
}: _*)
``` 

He suspected that it's expensive to call `withColumn` for a thousand times. Hence, he dived into its implementation and found out the above in the private method `withColumns` called by `withColumn`. In his fast version, only one new `DataFrame` was created.

I wondered why there was a significant cost difference and looked further into it. After turning on the debug log, I saw a lot of `=== Result of Batch Resolution ===`s in my slow version. It suddenly struck me that Catalyst's analysis might not be free. A thousand `withColumn`s were actually a thousand times of analysis, which held true for all APIs on `DataFrame`. On the other hand, analysis of transform on `Column` was actually lazy. 

The log led me to `org.apache.spark.sql.catalyst.rules.RuleExecutor` where I  spotted a `timeMap` tracking time running specific rules. What's more exciting, the statistics was exposed through `RuleExecutor.dumpTimeSpent` which I could add to compare the costs in two versions. 

```scala
df.columns.foldLeft(df) { case (df, col) =>
  println(RuleExecutor.dumpTimeSpent())
  df.withColumn(col, df(col).cast(IntegerType))
}
println(RuleExecutor.dumpTimeSpent())

df.select(df.columns.map { col =>
  println(RuleExecutor.dumpTimeSpent())
  df(col).cast(IntegerType)
}: _*)
println(RuleExecutor.dumpTimeSpent())

```

As expected, the time spent increased for each `DataFrame#withColumn` while that stayed the same for `Column#cast`. It would take about 100ms for one round of analysis. (I have a table of time spent for all 56 rules in 100 `withColumn` calls in appendix if you are curious).

### Summary 

1. The hidden cost of `withColumn` is Spark Catalyst's analysis time. 
2. The time spent in Catalyst analysis is usually negligible but it will become an issue when there is a large number of transforms on `DataFrame`. It's not unusual for a model to have 1000 features which may require preprocessing.
3. Don't create a new `DataFrame` for each transform on `Column`. Create one at last with `DataFrame#select`


### Appendix

Rule | Nano Time
---- | ---------
org.apache.spark.sql.catalyst.analysis.Analyzer$FixNullability	|	489262230
org.apache.spark.sql.catalyst.analysis.Analyzer$ResolveGroupingAnalytics	|	243030776
org.apache.spark.sql.catalyst.analysis.TypeCoercion$PropagateTypes	|	143141555
org.apache.spark.sql.catalyst.analysis.Analyzer$ResolveDeserializer	|	97690381
org.apache.spark.sql.catalyst.analysis.ResolveCreateNamedStruct	|	87845664
org.apache.spark.sql.catalyst.analysis.Analyzer$ResolveWindowFrame	|	85098172
org.apache.spark.sql.catalyst.analysis.Analyzer$ResolveWindowOrder	|	83967566
org.apache.spark.sql.catalyst.analysis.Analyzer$ExtractWindowExpressions	|	63928074
org.apache.spark.sql.catalyst.analysis.Analyzer$ResolveReferences	|	56549170
org.apache.spark.sql.catalyst.analysis.Analyzer$LookupFunctions	|	52411767
org.apache.spark.sql.catalyst.analysis.Analyzer$ExtractGenerator	|	24759815
org.apache.spark.sql.catalyst.analysis.ResolveTimeZone	|	24078761
org.apache.spark.sql.catalyst.analysis.Analyzer$ResolvePivot	|	23264984
org.apache.spark.sql.catalyst.analysis.ResolveInlineTables	|	22864548
org.apache.spark.sql.execution.datasources.FindDataSourceTable	|	22127481
org.apache.spark.sql.catalyst.analysis.DecimalPrecision	|	20855512
org.apache.spark.sql.catalyst.analysis.TypeCoercion$ImplicitTypeCasts	|	19908820
org.apache.spark.sql.catalyst.analysis.TimeWindowing	|	17289560
org.apache.spark.sql.catalyst.analysis.TypeCoercion$DateTimeOperations	|	16691649
org.apache.spark.sql.catalyst.analysis.TypeCoercion$FunctionArgumentConversion	|	16645812
org.apache.spark.sql.catalyst.analysis.ResolveHints$ResolveBroadcastHints	|	16391773
org.apache.spark.sql.catalyst.analysis.Analyzer$ResolveFunctions	|	16094905
org.apache.spark.sql.catalyst.analysis.TypeCoercion$InConversion	|	15937875
org.apache.spark.sql.catalyst.analysis.TypeCoercion$PromoteStrings	|	15659420
org.apache.spark.sql.catalyst.analysis.TypeCoercion$IfCoercion	|	15131194
org.apache.spark.sql.catalyst.analysis.TypeCoercion$BooleanEquality	|	15120505
org.apache.spark.sql.catalyst.analysis.TypeCoercion$Division	|	14657587
org.apache.spark.sql.execution.datasources.PreprocessTableCreation	|	12421808
org.apache.spark.sql.catalyst.analysis.Analyzer$ResolveSubquery	|	12330915
org.apache.spark.sql.catalyst.analysis.UpdateOuterReferences	|	11919954
org.apache.spark.sql.catalyst.analysis.TypeCoercion$CaseWhenCoercion	|	11807169
org.apache.spark.sql.catalyst.analysis.EliminateUnions	|	11761260
org.apache.spark.sql.catalyst.analysis.SubstituteUnresolvedOrdinals	|	11683297
org.apache.spark.sql.catalyst.analysis.ResolveHints$RemoveAllHints	|	11363987
org.apache.spark.sql.execution.datasources.DataSourceAnalysis	|	11253060
org.apache.spark.sql.catalyst.analysis.Analyzer$HandleNullInputsForUDF	|	11075682
org.apache.spark.sql.execution.datasources.PreprocessTableInsertion	|	11061610
org.apache.spark.sql.catalyst.analysis.Analyzer$GlobalAggregates	|	10708386
org.apache.spark.sql.catalyst.analysis.CleanupAliases	|	9447785
org.apache.spark.sql.catalyst.analysis.Analyzer$ResolveAliases	|	4725210
org.apache.spark.sql.catalyst.analysis.Analyzer$ResolveNewInstance	|	3634067
org.apache.spark.sql.catalyst.analysis.Analyzer$CTESubstitution	|	2359406
org.apache.spark.sql.catalyst.analysis.Analyzer$ResolveOrdinalInOrderByAndGroupBy	|	2191643
org.apache.spark.sql.catalyst.analysis.ResolveTableValuedFunctions	|	2160003
org.apache.spark.sql.catalyst.analysis.Analyzer$ResolveRelations	|	2095181
org.apache.spark.sql.catalyst.analysis.Analyzer$ResolveAggregateFunctions	|	2029468
org.apache.spark.sql.catalyst.analysis.TypeCoercion$WidenSetOperationTypes	|	1999994
org.apache.spark.sql.execution.datasources.ResolveSQLOnFile	|	1891759
org.apache.spark.sql.catalyst.analysis.Analyzer$ResolveMissingReferences	|	1864083
org.apache.spark.sql.catalyst.analysis.Analyzer$ResolveNaturalAndUsingJoin	|	1856631
org.apache.spark.sql.catalyst.analysis.Analyzer$ResolveAggAliasInGroupBy	|	1740242
org.apache.spark.sql.catalyst.analysis.Analyzer$ResolveGenerate	|	1714332
org.apache.spark.sql.catalyst.analysis.Analyzer$ResolveUpCast	|	1686660
org.apache.spark.sql.catalyst.analysis.Analyzer$PullOutNondeterministic	|	1602061
org.apache.spark.sql.catalyst.analysis.Analyzer$WindowsSubstitution	|	1406648
org.apache.spark.sql.catalyst.analysis.AliasViewChild	|	1184166




