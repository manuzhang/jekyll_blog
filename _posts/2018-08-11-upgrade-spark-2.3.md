---
comments: true
layout: post
title: Upgrading to Spark 2.3.1
--- 

Recently we upgraded our Spark cluster from 2.2.0 to 2.3.1. The transition has been smooth except that the Spark driver address has changed from IP to hostname. 

One thing I found immediately was that submitted application didn't run and I also failed to open application's Web UI page. I looked into the log and it said the driver couldn't be connected by its hostname, which is expected in our cluster environment. Back in Spark 2.2.0, executors found driver by its IP. So what has changed ? Google led me to [SPARK-21642](https://issues.apache.org/jira/browse/SPARK-21642) whose change set driver address to FQDN from IP for SSL cases. This would break in environments where hostname could not be resolved. Luckily, a nice guy provided a solution in the comment

> I ran into the same problem. I simply put `export SPARK_LOCAL_HOSTNAME=$(ip route get 1 | awk '{print $NF;exit}')`
 into my launch script to solve the issue. This forces spark to use your IP as hostname. It should work on any debian based system.
 
We set `SPARK_LOCAL_HOSTNAME` to IP in `spark-env.sh` and it worked like a charm.
