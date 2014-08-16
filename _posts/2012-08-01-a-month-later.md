---
author: manuzhang
comments: true
date: 2012-08-01 04:03:47+00:00
layout: post
slug: a-month-later
title: A month later...
wordpress_id: 261
categories:
- database
- 实习笔记
tags:
- Cassandra
- nosql
---

It's already a month after I started my internship. Let's see how it is going.



It began with MonetDB and Eddies, and my major concern has since then been on In-Memory Database and Adaptive Query Processing (AQP). Sadly, MonetDB, the Column-Store pioneer, didn't go well with Eddies, the cornerstone in AQP because of the per-tuple basis of the latter. Hence, we substituted Cassandra for MonetDB. The reason we stick with Eddies seemed to be that we'd got no other choices while it didn't take long to find another open-source database system. Nonetheless, I did learn the concepts of column-store, DSM and radix cluster from MonetDB.



<!-- more -->

That was my first touch with NoSQL systems. It appeared to me that everyone is promoting NoSQL Systems now. So what made NoSQL outweigh RDBMS? All I'd learned were short terms like Big Data, Scalability and Availability where NoSQL could be a better solution. Then, I got a chance to check out which performed better in real application.



The thing is that we had several GB of data which were originally stored in [Hive](http://hive.apache.org/), and we intended to run queries with joins and aggregations. Although Hive provides a SQL-like language for those who has no experience in writing [MapReduce](http://en.wikipedia.org/wiki/MapReduce) programs, the underlying system is [Hadoop](http://en.wikipedia.org/wiki/Hadoop) and all the queries are executed through MapReduce. Hence, it's a NoSQL architecture. What's disheartening was most queries took so long that the project was unlikely to finish ahead of deadline (Well, this is my personal guess because it's before I got involved in but otherwise there would be no point to bring me in).



The rest of story was that we gave a try on [Greenplum Database](http://www.greenplum.com/products/greenplum-database), and it surpassed Hive by a considerable amount. Hence, we decided to migarate our project to Greenplum. Greenplum Database is an RDBMS (on top of [PostgreSQL](http://en.wikipedia.org/wiki/Postgre)) and more than an RDBMS:





<blockquote>
  Built to support Big Data Analytics, Greenplum Database manages, stores, and analyzes Terabytes to Petabytes of data. Users experience 10 to 100 times better performance over traditional RDBMS products – a result of Greenplum’s shared-nothing MPP architecture, high-performance parallel dataflow engine, and advanced gNet software interconnect technology.


</blockquote>



This is how it looks like. A master host storing meta data (like schema) and the tables are distribued among the segment hosts. Typically, a scan query will be executed parallelly in all the segment hosts. Hence, Greenplum could easliy fit into a shared-nothing cluster. The enterprise version would cost us a fortune so we got by with the community version which could only be installed on one node. Despite that, we were able to utilize our 8-core machines.



![Greenplum architecture](https://lh3.googleusercontent.com/-r-6ZtddyLEA/UBYxMM1IJuI/AAAAAAAAAak/UNgijBV1lFY/w367-h276-n-k/Screenshot%2Bfrom%2B2012-07-30%2B15%253A00%253A25.png)



The takeaway is that we can't simply state NoSQL trumps RDBMS or vice versa. Firstly, our pain only shows that this is an unsuitable task for MapReduce model, which might be a better solution faced with another problem. Also, we didn't try out other NoSQL Systems (e.g. Cassandra). On the other hand, we can not regard Greenplum as a traditional RDBMS. It has evolved to be eligible in a distributed environment and deal with Big Data elegantly. One thing that hasn't changed is our RDBMS is expensive while there is a bunch of open-source NoSQL projects. I may have more say on this topic later but let's get back to Cassandra.



Here's a visual guide to NoSQL Systems.



![NoSQL Systems](https://lh3.googleusercontent.com/-3DKHTfdV7Es/UBiEWtZ8YDI/AAAAAAAAAew/x6u2lbT51hA/w480-h359-n-k/media_httpfarm5static_mevik.png.scaled500.png)



The picture visualizes the so called CAP theorem posited by [Eric Brewer](http://www.cs.berkeley.edu/~brewer/). So what C, A, P stand for respectively and what the theorem tells us?





<blockquote>
  Consistency


  
  
> 
> <blockquote>
    All database clients will read the same value for the same query, even given con- current updates.


  </blockquote>
> 
> 
  
  Availability


  
  
> 
> <blockquote>
    All database clients will always be able to read and write data.


  </blockquote>
> 
> 
  
  Partition Tolerance


  
  
> 
> <blockquote>
    The database can be split into multiple machines; it can continue functioning in the face of network segmentation breaks.


  </blockquote>
> 
> 
  
  Brewer’s theorem is that in any given system, you can strongly support only two of the three. This is analogous to the saying you may have heard in software development: “You can have it good, you can have it fast, you can have it cheap: pick two.”


</blockquote>



We can see where Greenplum and Cassandra lie in the continuum. At the first sight of the theorem, I thought I'd have to trade off, to sacrifice one to support the other two. But remark the word "strongly". As it is shown in Cassandra, **eventual consistency** doesn't mean inconsistency. For a better understanding, please refer to Eric Brewer's recent [article](http://www.infoq.com/articles/cap-twelve-years-later-how-the-rules-have-changed), _CAP Twelve Years Later: How the "Rules" Have Changed_.



My following posts will mainly cover various aspects of Cassandra. I'm reading the [Definitive Guide](http://www.ppurl.com/2010/12/cassandra-the-definitive-guide.html) as well as source codes.



