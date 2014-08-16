---
author: manuzhang
comments: true
date: 2013-01-02 16:01:02+00:00
layout: post
published: false
slug: cassandra-gossiper
title: Cassandra -- Gossip
wordpress_id: 435
categories:
- database
- 实习笔记
tags:
- Cassandra
---

As said in [Cassandra -- partitioning](http://ifthiskills.me/?p=773), each Cassandra peer tracks all other peers (a peer is a physical node). Cassandra is a decentralized database and there is no global monitor to synchronize the cluster's membership. To solve the problem, Cassandra uses [gossip protocol](http://en.wikipedia.org/wiki/Gossip_protocol) and a peer's state _eventually_ propagates to all other peers. 

Moreover, each data item is replicated to a set of nodes. The set of nodes have to periodically synchronize with each other such that all replicas will _eventually_ become consistent. [ReadRepair](http://wiki.apache.org/cassandra/ReadRepair) could help more or less by pushing the most recent version to any out-of-date replicas involved in read but what if read rarely happens and mostly from one node? There is no guarantee for eventual consistency. The problem is addressed by [Anti-entropy](http://wiki.apache.org/cassandra/ArchitectureAntiEntropy), which is more expensive than gossip and should be used more infrequently.

As per Wikipedia, the paper by Demers[1] is considered by most researchers to be the first to have really recognized the power of these protocols and to propose a formal treatment of gossip.

The paper examined such methods as _Direct mail_ (not entirely reliable), Anti-entropy (extremely reliable but requires examining the contents of the database) and Rumor mongering (similar to gossip in Cassandra) for spreading updates. Further, to remedy the problem that propagation mechanism will spread old copies of the item from elsewhere in the database back to the site where they have deleted it, the authors replace deleted items with _ death certificates _ which carry timestamps and spread like ordinary data. The death certificates are called [tombstones](http://wiki.apache.org/cassandra/DistributedDeletes) in Cassandra.

<!-- more -->

Now let me introduce Gossip[2] and Anti-entropy in Cassandra respectively.



# Gossip



Cassandra uses the _push-pull gossip approach_ which has the best efficiency (best convergence) in comparison with push or pull as examined in [Distributed Algorithms in NoSQL Databases](http://highlyscalable.wordpress.com/2012/09/18/distributed-algorithms-in-nosql-databases/) (the article is highly recommended).



![](https://lh3.googleusercontent.com/-pJ9EFd2Ry4M/UOQ-gSF5iWI/AAAAAAAAA0U/4mkw0L9Uemo/w247-h212-n-k/Screenshot%2Bfrom%2B2013-01-02%2B22%253A01%253A25.png)



Suppose we have two peers A@10.0.0.1 and B@10.0.0.2 gossiping (as shown in the graph) and without loss of generality A starts the chat. Firstly, A tells B what it knows about others and itself via _GossipDigestSynMessage_. Receiving the message (handled by _GossipDigestSynVerbHandler_), B compares the info with what it already gets (by generation and version, talked later). Then, B replies with what A has yet known and asks for the part A knows but B doesn't via _GossipDigestAckMessage_. Hearing that (handled by _GossipDigestAckVerbHandler_), A updates its own info and continues with what B needs. Finally, B updates (handled by _GossipDigestAck2VerbHandler_) its knowledge and synchronizes its view of the world with A. 




## Seeds


[cci] org.apache.cassandra.gms.Gossiper [/cci] maintains a peer's relationship with other peers.  It is started before the peer joins the ring and the peer has to join the ring to know the other guys (except for the first one). To solve the chicken and egg problem, Cassandra administrator will have to firstly set the _seeds_ in [cci]conf/cassandra.yaml[/cci] (the default is localhost):

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
          - seeds: "127.0.0.1"
[/cc]



## Gossip rule






  1. gossip to random live peer


  2. gossip to random unreachable peer with some probability


  3. 
    (i)if there are no live peers, gossip to random seed
    (ii)if the peer gossiped to was not seed or _the live peers are less than seeds_, gossip to random seed with some probability









## EndpointState



Gossiper maintains a map from each peer to its associated _EndpointState_.
[cc lang="java"]
// org.apache.cassandra.gms.Gossiper
final ConcurrentMap endpointStateMap = new ConcurrentHashMap();
[/cc]

_EndpointState_ is comprised of HeartBeatState and ApplicationState.

_HeartBeatState_ further consists of _generation number_ and _version number_.

The generation number, stored in the _SystemTable_，stays the same when server is running and grows every time it starts (also grows when changing tokens). "Current time" serves as the generation number in most cases. (Refer to [Anatomy of a Cassandra partition](http://thelastpickle.com/2011/12/15/Anatomy-of-a-Cassandra-Partition/) for exception and solutions)

When the Gossiper starts, it retrieves its generation number from SystemTable, stores the seed sets, initializes the heartbeat state for the localEndpoint and schedules gossip task (a peer gossips every second).

The version number is a counter for a peer's heartbeat. It's increased every gossip round so that the _FailureDetector_ (talked next) could decide whether the peer is still alive.

_ApplicationState_ tells more about state if HeartBeatState signifies state changes. An ApplicationState is essentially a key/value pair. For example, "STATUS" has values "BOOT", "NORMAL", "LEAVING", etc. (see [cci]org.apache.cassandra.gms.VersionedValue[/cci])

[cc lang="java"]
// org.apache.cassandra.gms.ApplicationState
public enum ApplicationState
{
    STATUS,
    LOAD,
    SCHEMA,
    DC,
    RACK,
    RELEASE_VERSION,
    REMOVAL_COORDINATOR,
    INTERNAL_IP,
    RPC_ADDRESS,
    X_11_PADDING, // padding specifically for 1.1
    SEVERITY,
    NET_VERSION,
    HOST_ID,
    TOKENS,
    // pad to allow adding new states to existing cluster
    X1,
    X2,
    X3,
    X4,
    X5,
    X6,
    X7,
    X8,
    X9,
    X10,
}
[/cc]




## FailureDetector


The _FailureDetector_ sits on top of Gossip protocol and is an implementation of [The Phi Accrual Failure Detector by Hayashibara et.al.](http://ddg.jaist.ac.jp/pub/HDY+04.pdf)



<blockquote>
Instead of providing information of a boolean nature (true vs. suspect), accrual failure detectors output a suspicion level on a continuous scale.

The particularity of the [latex]\varphi[/latex] failure detector is that it dynamically adjusts to current network conditions the scale on which the suspicion level is expressed.
</blockquote>



The figure below illustrates how the [latex]\varphi[/latex] failure detector works (from the original paper):


![](https://lh5.googleusercontent.com/-uSiqFNjBcXc/UOaDyP8-AVI/AAAAAAAAA0o/k1r4ULtqqfk/w430-h254-n-k/Screenshot%2Bfrom%2B2013-01-04%2B15%253A22%253A39.png)



Gossiper reports heartbeat of a peer to FailureDetector on generation change or version change when receiving one of the three messages talked previously and convicts the peer when [latex]\varphi \geq \Phi[/latex] where [latex]\Phi[/latex](phi_convict_threshold) is default 8.

The phi_convict_threshold is configurable in [cci]conf/cassandra.yaml[/cci] and should be in the range [latex][5, 16][/latex]:
[cc lang="yaml"]
# phi value that must be reached for a host to be marked down.
# most users should never need to adjust this.
# phi_convict_threshold: 8
[/cc]



## Live or Dead


Gossiper remembers whether a certain peer is alive, unreachable, left or removed. It keeps the memory in the following four containers besides endpointStateMap and _doStatusCheck_ after sending out all GossipDigestSyn messages.
[cc lang="java"] 
// org.apache.cassandra.gms.Gossiper

private final Set liveEndpoints = new ConcurrentSkipListSet(inetcomparator);

private final Map unreachableEndpoints = new ConcurrentHashMap();

private final Map expireTimeEndpointMap = new ConcurrentHashMap();

private final Map justRemovedEndpoints = new ConcurrentHashMap();

Gossiper considers a peer alive and adds it to the liveEndpoints. when receiving GossipDigestAct or GossipDigestAct2 and the peer's state is not _dead_ (DEAD_STATES as follows)

[cc lang="java"]
// org.apache.cassandra.gms.Gossiper
static final List DEAD_STATES = Arrays.asList(VersionedValue.REMOVING_TOKEN, VersionedValue.REMOVED_TOKEN,
                                                          VersionedValue.STATUS_LEFT, VersionedValue.HIBERNATE);

[/cc]

A peer is convicted unreachable and thus put into unreachableEndpoints if its [latex]\varphi[/latex] value exceeds the phi_convict_threshold. 

A peer who is decommissioning or being removed will advertise its states is thrown into the expireTimeEndpoint (we do not remove it from liveEndpoints). Gossiper wait for _aVeryLongTime_ before removing its memory of the peer completely (only wait for 30 seconds for a client_only peer). We can not remove the EndpointState immediately since the peer which hasn't received the news may propagate back a live state back to us. We would take it as a new peer as its EndpointState is already deleted. 

[cc lang="java"]
public static final long aVeryLongTime = 259200 * 1000; // 3 days
[/cc]

Finally, the QUARANTINE_DELAY (1 minute by default, during which no state changes about the justRemovedEndpoints are served) further prevents peers from falsely reincarnate.

I've drawn a graph to illustrate the lifetime of an Endpoint.



![](https://lh5.googleusercontent.com/-vcsX_3NnOzs/UOfgubeYyNI/AAAAAAAAA1A/u2uOA79QW08/w213-h276-n-k/doStatusCheck.jpg)





# Anti-entropy







# References 






  1. [Epidemic algorithms for replicated database maintenance. Alan Demers, et al. Proc. 6th ACM PODC, Vancouver BC, 1987](http://dl.acm.org/citation.cfm?id=41841)


  2. [ArchitectureGossip](http://wiki.apache.org/cassandra/ArchitectureGossip)

