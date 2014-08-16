---
author: manuzhang
comments: true
date: 2012-11-03 02:02:09+00:00
layout: post
slug: cassandra-revisited
title: Cassandra Revisited
wordpress_id: 708
categories:
- database
tags:
- Cassandra
---

一直在读 Cassandra 的源码。当初计划的是把对源码的解释写到博客上，也写了几篇（大部分都还处于草稿阶段），但是效果不理想。包含大段代码的文章可读性太差，即便是自己要回忆一些细节，也是云里雾里的。另一方面，源码随着 Cassandra 版本的更新会不断变化，时效性很差。因此，我想把这一系列的文章重新写过，少一些代码，多一些图片，重要的是把问题解释清楚（如 bootstrap，write / read quorum，hinted handoff 等等）。<strike>最后，换成中文表达</strike>。



这里先列个提纲吧。





# entry







  * Cassandra Daemon


  * StorageServices





# operation







  * bootstrap / decommission (unbootstrap)    


  * write (hinted handoff) 


  * read (read repair)


  * compaction





# data model







  * commit log


  * memtable (keyspace / column family / column)


  * SSTable


  * cache (KeyCache / RowCache)


  * Token / Range





# architecture







  * partitioner


  * replica placement strategy / snitch


  * consistency level


  * message service


  * gossip (failure detection)


  * streaming





# interface







  * StorageProxy


  * CLI / nodetool


  * Thrift API



这些都是我现在接触到的，让我有疑惑的。我不会再特地去研究某个模块或者类的代码了，所以这里先不涉及 （如 CQL），将来用到了再补充。



