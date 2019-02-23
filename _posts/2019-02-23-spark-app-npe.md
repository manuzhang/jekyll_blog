---
comments: true
layout: post
title: NPE from Spark App that extends scala.App
--- 

This is the log of a bizarre `NullPointerException(NPE)` from a Spark application that extends `scala.App` and a minor bug of Spark hidden for years. 

Last week, I was developing the following Spark (2.3.1) application that pushes data from `Hive` to a database.

```scala
object Test extends App {
  val foo = args(0)
  val bar = args(1)
  
  val spark = SparkSession.builder
    .getOrCreate()
    
  spark.sql("select * from test")
    .foreachPartition { rowIter: Internal[Row] =>
       val client = new DbClient
       client.setFoo(foo) // throws NullPointerException
       client.setBar(bar)
       client.init
      
       client.put(rowIter.map { row =>
         row.getLong(0) -> row.getLong(1)
       }.toMap)
    }
}
```

The local variables `foo` and `bar` are used to configure the `DbClient` on remote executors. I was expecting them to be shipped through [closures](https://spark.apache.org/docs/latest/rdd-programming-guide.html#understanding-closures-) but somehow they weren't. 

My colleague Vincent pointed me to `ClosureCleaner` which `logDebug` [all fields, methods and classes](https://github.com/apache/spark/blob/v2.3.1/core/src/main/scala/org/apache/spark/util/ClosureCleaner.scala#L221) in closures. I enabled driver's debug log and the absence of `foo` and `bar` assured me that they were not captured by closure. 

Then I looked into the byte code of what's executed in `foreachPartition`. It reminded me that `foo` and `bar` were static members of `Test$` rather than local variables.
 
```bash
      9: getstatic      #26                 // Field Test$.MODULE$:LTest$;
      12: invokevirtual #30                 // Method Test$.foo:()Ljava/lang/String;
      15: invokevirtual #34                 // Method DbClient.setFoo:(Ljava/lang/String;)V
```

**It suddenly struck me that the initialization of the class (`Test$`) that extends `scala.App` is actually delayed to its `main()` method**. That's why `foo` and `bar` were uninitialized.

Meanwhile, I googled "Spark closure problems" which led me to an ancient Spark jira [Closure problems when running Scala app that "extends App"](https://issues.apache.org/jira/browse/SPARK-4170). There was a fix that would print a warning when an application extends `App`. 

```scala
      if (classOf[scala.App].isAssignableFrom(mainClass)) {
        printWarning("Subclasses of scala.App may not work correctly. Use a main() method instead.")
      }
```
Why have I never been warned ? After some digging, I realized the above "if clause" would never be true. This is because Scala compiler generate two Java classes `Test` and `Test$` from `object Test` and the `mainClass` (`Test`) I passed in is not that (`Test$`) extends `scala.App`. It's recorded in [SPARK-26977](https://issues.apache.org/jira/browse/SPARK-26977).

The solution is to override `main` method of `scala.App` and put everything in it. Then `foo` and `bar` are initialized local variables and shipped remotely through closure.

### Appendix
1. How to decompile the anonymous function passed to `foreachPartition` ?

  ```
  javap -c Test\$\$anonfun\$1 
  ```
  
2. How to enable driver's debug log ?

   Prepare a `log4j.properties` file with 
   
   ```
   log4j.rootLogger=DEBUG
   ```
   
   and submit a Spark application with
   
   ```
   --driver-java-options "-Dlog4j.configuration=file:/path/to/log4j.properties"
   ```