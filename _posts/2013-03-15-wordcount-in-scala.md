---
author: manuzhang
comments: true
date: 2013-03-15 09:27:21+00:00
layout: post
slug: wordcount-in-scala
title: WordCount in Scala
wordpress_id: 1068
categories:
- 实习笔记
tags:
- scala
---

[Reynold S. Xin](http://www.cs.berkeley.edu/~rxin/) from AMPLab, UCB is visiting IMC and giving courses on Spark and Shark.  


<blockquote>
Spark is an open source cluster computing system that aims to make data analytics fast — both fast to run and fast to write.
</blockquote>


So basically we could do MapReduce-like job in Spark. Probably our codes will be more intuitive and also run faster than Hadoop. Here's the WordCount example:
[cc lang="scala"]
file = spark.textFile("hdfs://...")
 
file.flatMap(line => line.split(" "))
    .map(word => (word, 1))
    .reduceByKey(_ + _)
[/cc]

<!-- more -->

I went to the course today and we got out feet wet with Spark this afternoon. ([Here](http://www.cs.berkeley.edu/~rxin/ampcamp-ecnu/) are the exercises). As a prerequisite for Spark shell (which is built on top of Scala shell), we firstly got some experience with Scala.

Of course, we did a WordCount exercise in Scala shell.
In the above WordCount, the [cci lang="scala"]textFile[/cci] and [cci lang="scala"]reduceByKey[/cci] is native in Spark but not in Scala. So we have to implement it ourselves and that's where I got stuck **for hours**. 

The exercise gave out how to load lines from a document and I started from there.
[cc lang="scala"]
import scala.io.Source
scala> val lines = Source.fromFile("/home/imdb_1/spark/spark-0.7.0/README.md").getLines.toArray
[/cc]

Then to split a line into words and put the output arrays of words in a single array, I used [cci lang="scala"]flatMap[/cci] as above. 

[cc lang="scala"]
val words = line.flatMap(line => line.split("\\s+"))
[/cc]

Now wrap each word in a pair with a count of 1. 

[cc lang="scala]
val map_output = word.map(word => (word, 1))
[/cc]

Like the shuffle and sort periods in MapReduce, I grouped the array of (word, count) pairs by word. The output is a map from word to array of (word, count) pairs. 

[cc lang="scala"]
val before_reduce = map_output.groupBy(_._1)
[/cc]

The final step was to get a new map which is from word to count of word.
[cc lang="scala"]
val reduce_output = before_reduce.map(kv => (kv._1, kv._2.foldLeft(0)((sum, v) => sum + v._2)))
[/cc]

In a pure functional programming, there are only immutables (Scala is not a pure one). If you are thinking about a for-loop here (although legitimate in Scala), try recursion, map, reduce or fold (Having had some experiences in Haskell, I still could not get the hang of them). 

Ok, put it together and I got oneline WordCount in Scala. 
[cc lang="scala"]
Source.fromFile("/home/imdb_1/spark/spark-0.7.0/README.md").getLines.toArray
      .flatMap(line => line.split("\\s+"))
      .map(word => (word, 1))
      .groupBy(_._1).map(kv => (kv._1, kv._2.foldLeft(0)((sum, v) => sum + v._2)))
[/cc]

Now, how the solution does it. 
[cc lang="scala"]
import scala.io.Source

val  lines = Source.fromFile("/home/imdb_1/spark/spark-0.7.0/README.md").getLines.toArray
val counts = new collection.mutable.HashMap[String, Int].withDefaultValue(0)
lines.flatMap(line => line.split(" ")).map(word => counts(word) += 1)
[/cc]
Simple and embarrassing. **Why do I have to implement it in a MapReduce way?** 

