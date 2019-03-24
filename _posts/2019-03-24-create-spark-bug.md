---
comments: true
layout: post
title: Fixing one Spark bug lead to another
--- 

As we mentioned in the [last post](https://manuzhang.github.io/2019/02/23/spark-app-npe.html), Spark had a bug checking whether an application has extended `scala.App` ([SPARK-26977](https://issues.apache.org/jira/browse/SPARK-26977)). I went on to submit a [pull request](https://github.com/apache/spark/pull/23903) checking the existence of `childMainClass$` on `spark-submit --class childMainClass`.

```scala
      Try {
        if (classOf[scala.App].isAssignableFrom(Utils.classForName(s"$childMainClass$$"))) {
          logWarning("Subclasses of scala.App may not work correctly. " +
            "Use a main() method instead.")
        }
      }
      // invoke main method of childMainClass
``` 

This was a minor issue but since it's my first PR to Spark I was quite happy about it. I didn't realize then when fixing one minor Spark bug I created a major one. 

That's [SPARK-27205](https://issues.apache.org/jira/browse/SPARK-27205), where launching `spark-shell` with `--packages` option failed to load transitive dependencies on master.

```
./bin/spark-shell --packages org.apache.spark:spark-sql-kafka-0-10_2.11:2.4.0
```

Weirdly, it would work after [removing my changes](https://github.com/apache/spark/pull/24147). The two looked irrelevant so I dug into how `spark-shell` load dependencies behind the scene.

`spark-shell` is actually a shorthand for `spark-submit --class org.apache.spark.repl.Main` and [repl.Main](https://github.com/apache/spark/blob/master/repl/src/main/scala/org/apache/spark/repl/Main.scala) loads user specified jars or packages from `spark.repl.local.jars` option of `SparkConf`. The option is set with `--jars` or `--packages` from `SparkSubmit`. I verified the option existed with user packages at `SparkSubmit`'s side so the issue was that somehow `SparkConf` didn't get to `repl.Main`. 

It turns out that all options of `SparkConf` are [set to system properties right before invoking the main method](https://github.com/apache/spark/blob/master/core/src/main/scala/org/apache/spark/deploy/SparkApplication.scala#L47) of `repl.Main` and loaded back into `SparkConf` at initialization of `repl.Main`. The latter has to **happen after** the former while "my fix" broke it ! 

Okay, I have been bitten twice by fields initialization of Scala Object.

