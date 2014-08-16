---
author: manuzhang
comments: true
date: 2012-10-15 13:52:24+00:00
layout: post
published: false
slug: cassandra-codebase-bootstrapping
title: Cassandra -- bootstrapping
wordpress_id: 392
categories:
- database
- 实习笔记
tags:
- Cassandra
---

Although titled "bootstrapping", this post is not only about _bootstrapping_ but all things _joinTokenRing_ ([cci]org.apache.cassandra.service.StorageService.joinTokenRing[/cci]).  In other words, what happens if we start up a Cassandra process with [cci lang="bash"]bin/cassandra -f[/cci]?

There could be a couple of scenarios so I drew a flowchart to present an overview.
<!-- more -->


I will walk through them and mention configuration issues along the way.



# New Cluster


When launching a new Cassandra cluster with first set of peers, you probably follow the red routes in the following flowchart. 



_auto_bootstrap_ is hard-coded true and the option no longer exists in [cci]conf/cassandra.yaml[/cci] (I believe we could add the option and overwrite the default value).

The first set must include a seed for later peers to join the ring. The seeds will specify the seeds option with IP addresses of their own and other seeds (so that seeds could see each other and avoid partition):

[cc lang="yaml"]
# any class that implements the SeedProvider interface and has a
# constructor that takes a Map of parameters will do.
seed_provider:
    # Addresses of hosts that are deemed contact points. 
    # Cassandra nodes use this list of hosts to find each other and learn
    # the topology of the ring.  You must change this if you are running
    # multiple nodes!
    - class_name: org.apache.cassandra.locator.SimpleSeedProvider
      parameters:
          # seeds is actually a comma-delimited list of addresses.
          # Ex: ",,"
          - seeds: "192.168.1.112"
[/cc]

Since it's first launch, there are no saved_tokens. You may specify _initial_token_ or not. In the latter case, tokens are generated randomly which could lead to unbalanced ring. 

[cc lang="yaml"]
# If you haven't specified num_tokens, or have set it to the default of 1 then
# you should always specify InitialToken when setting up a production
# cluster for the first time, and often when adding capacity later.
# The principle is that each node should be given an equal slice of
# the token ring; see http://wiki.apache.org/cassandra/Operations
# for more details.
#
# If blank, Cassandra will request a token bisecting the range of
# the heaviest-loaded existing node.  If there is no load information
# available, such as is the case with a new cluster, it will pick
# a random token, which will lead to hot spots.
initial_token:
[/cc]



# Bootstrap


As per [CassandraWiki](http://wiki.apache.org/cassandra/Operations#Bootstrap):


<blockquote>
Adding nodes is called bootstrapping.
</blockquote>


Here we specifically mean adding those non-seed nodes which will finally trigger the [cci]org.apache.cassandra.service.StorageService.bootstrap[/cci] method as highlighted by the green route in the flowchart.



You could calculate and specify _initial_token_ or let the system randomly generate tokens. Another option you want to set is _num_tokens_. The new feature is introduced into **Cassandra 1.2** as _Virtual Nodes_[1][2][3].

[cc lang="yaml"]
# This defines the number of tokens randomly assigned to this node on the ring
# The more tokens, relative to other nodes, the larger the proportion of data
# that this node will store. You probably want all nodes to have the same number
# of tokens assuming they have equal hardware capability.
#
# If you leave this unspecified, Cassandra will use the default of 1 token for legacy compatibility,
# and will use the initial_token as described below.
#
# Specifying initial_token will override this setting.
#
# If you already have a cluster with 1 token per node, and wish to migrate to 
# multiple tokens per node, see http://wiki.apache.org/cassandra/Operations
# num_tokens: 1
[/cc]

The default value is 1 for legacy compatibility while 256 is recommended. The subtlety here is when num_tokens is 1 Cassandra will pick a token that half the load of most-loaded mode otherwise we just move on to _bootstrap_ (load is more likely to be balanced across the ring).



# Restart


_Restart_ here applies to occasions when machines restart after failure, restart for migration, or restart after decommission, as marked by the blue route.



Since we've already bootstrapped before, we have stored saved_tokens and restart from the previous position on the ring. 

We should pay attention to migration here (which means you specify the _num_tokens_ value greater than 1). It's a two-step process and I've drawn graphs to help you understand. 

Say we have four peers, each configured with [cci lang="yaml"]num_tokens:1[/cci]. This may be how our original cluster look like. (That machine is responsible for keys partitioned into green parts)



![](https://lh4.googleusercontent.com/-kNHu4IasWJQ/UO4zFK9l3nI/AAAAAAAAA4E/3rcx1Sfywso/w298-h263-n-k/Screenshot%2Bfrom%2B2013-01-10%2B11%253A16%253A56.png)





We notice one machine could handle more loads so we decide not to waste it and increase its _num_tokens_ value. We shutdown Cassandra on that machine, change the configuration to [cci lang="yaml"]num_tokens:6[/cci], and restart Cassandra. Are we done yet? No, check out the graph below. 



![](https://lh5.googleusercontent.com/-AV5nuvIcEyg/UO4zFae1rOI/AAAAAAAAA4A/DRStv5wB2hg/w289-h263-n-k/Screenshot%2Bfrom%2B2013-01-10%2B11%253A17%253A30.png)



Except that green parts are sliced into six sectors, no changes to loads each machine handles. Then we need to invoke [cci lang="bash"]bin/cassandra-shuffle[/cci] command to shuffle the tokens and complete the migration. The final ring could look like the following.   



![](https://lh4.googleusercontent.com/-Nl0ZHbSZWJs/UO4zFvIXgQI/AAAAAAAAA4I/a6ID1cTmWLk/w277-h263-n-k/Screenshot%2Bfrom%2B2013-01-10%2B11%253A18%253A20.png)



The green sectors spread out and consume larger areas on the ring. Now we are done and point more loads to one underlying physical nodes. _Accounting for hardware heterogeneity is one advantage for using virtual nodes_



# Replace a dead node


Our last scenario is when we replace a dead node with a new one with 
[cci lang="bash"]bin/cassandra -Dcassandra.replace_token=[/cci][4] starting from **Cassandra 1.0**.


Note that an [cci]UnsupportedOperationException[/cci] will be thrown if we try to replace a live node. Also, the token must already be a token on the ring. 

Check [CassandraWiki](http://wiki.apache.org/cassandra/Operations#Replacing_a_Dead_Node) for a detailed description.



# Reference






  1. [Virtual Nodes on Cassandra Summit 2012](https://www.youtube.com/watch?v=GddZ3pXiDys&list=PLC5E3906433F5A165&index=28)


  2. [Virtual node strategies](http://www.acunu.com/2/post/2012/07/virtual-nodes-strategies.html)


  3. [Improving Cassandra's uptime with virtual nodes](http://www.acunu.com/2/post/2012/10/improving-cassandras-uptime-with-virtual-nodes.html)


  4. [Cassandra utilities](http://www.datastax.com/docs/1.2/references/cassandra)


