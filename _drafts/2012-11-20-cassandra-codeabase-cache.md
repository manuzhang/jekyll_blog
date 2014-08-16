---
author: manuzhang
comments: true
date: 2012-11-20 08:52:37+00:00
layout: post
published: false
slug: cassandra-codeabase-cache
title: Cassandra -- Cache
wordpress_id: 449
categories:
- database
- 实习笔记
tags:
- Cassandra
---

Now we are talking about caches in Cassandra.

Cassandra offers built-in **KeyCache** and **RowCache**.

To get a feel of their positions in the Cassandra anatomy.



![](https://lh4.googleusercontent.com/-el-gyCU5oBk/UNrtefBETZI/AAAAAAAAAvg/oj_IG1GWXhc/w273-h272-n-k/Screenshot%2Bfrom%2B2012-12-26%2B20%253A20%253A53.png)



From [datastax documentation](http://www.datastax.com/dev/blog/caching-in-cassandra-1-1):


<blockquote>

> 
> 
	
>   * The **KeyCache** is essentially a cache of th7e primary key index for a Cassandra table. It will save CPU time and memory over relying on the OS page cache for this purpose. However, enabling just the key cache will still result in disk (or OS page cache) activity to actually read the requested data rows.
> 
	
>   * The **row cache** is more similar to a traditional cache like memcached: when a row is accessed, the entire row is pulled into memory (merging from multiple sstables, if necessary) and cached so that further reads against that row can be satisfied without hitting disk at all.
> 

</blockquote>



<!-- more -->

Here is a class diagram of caches in Cassandra (I tried out [Creately](creately.com) this time but the clarity doesn't seem to have improved a lot).


![cache class](https://lh4.googleusercontent.com/--Yj-v9FUCnU/UKRlfo0hOwI/AAAAAAAAAu4/iqka7-hrlRA/w432-h276-n-k/cache2.png")



A singleton class _CacheService_ (please refer to [cci]org.apache.cassandra.service.CacheService[/cci]) offers two caches, _keyCache_ and _rowCache_, both of which are _AutoSavingCache_. They are called so because they will be periodically saved to disks to solve the cold start problem (we will get to it later). The AutoSavingCache inherits _InstrumentingCache_ where we could find a bunch of "get" and "put" methonds. InstrumentingCache is a wrapper over a map. Be it _ConcurrentLinkedHashCache_ or _SerializingCache_ (we will introduce them when talking about RowCache), it is intrinsically a [ConcurrentLinkedHashMap](https://code.google.com/p/concurrentlinkedhashmap/)).




# KeyCache


[cc lang="java"]
// org.apache.cassandra.service.CacheService
public final AutoSavingCache keyCache;
[/cc]

KeyCache caches row key position in [cci]*-Data.db[/cci] file. It removes up to 1 disk seek per-SSTable (avoid that [cci]*-Index.db[/cci] access).

Check it out in [cci]org.apache.cassandra.db.RowIndexEntry[/cci].
[cc lang="java"]
public class RowIndexEntry
{
    public final long position;
    ...
}
[/cc]

KeyCache is global from the architecture view but **per-SSTable** from the function view.

Please jump to [cci]org.apache.cassandra.io.sstable.SSTableReader[/cci]:
[cc lang="java"]
private final InstrumentingCache keyCache = CacheService.instance.keyCache;
[/cc]
This shows that each SSTable keeps a reference to the global KeyCache. So how does a SSTable find its own KeyCache? By _KeyCacheKey_:
[cc lang="java"]
package org.apache.cassandra.cache
public class KeyCacheKey implements CacheKey
{
    public final Descriptor desc;
    public final byte[] key;
    ...
};
[/cc]
Hence, to be more exact, by its _Descriptor_. I won't go deeper here but just remember that an SSTable is described by Descriptor somehow.

We could configure the maximum size of the KeyCache in [cci]conf/cassandra.yaml[/cci]

[cc lang="yaml"]
# Default value is empty to make it "auto" (min(5% of Heap (in MB), 100MB)). Set to 0 to disable key cache.
key_cache_size_in_mb:
[/cc]
The "auto" size is the lesser of 5% of Heap and 100MB and thus the KeyCache is enabled by default.

Recall that we have two specific types of caches (from the data structure view), i.e. _ConcurrentLinkedHashCache_ and _SerializingCache_. KeyCache is a ConcurrentLinkedHashCache.

[cc lang="java" highlight="9,21"]
// org.apache.cassandra.service.CacheService
    private AutoSavingCache initKeyCache()
    {
        ...
        long keyCacheInMemoryCapacity = DatabaseDescriptor.getKeyCacheSizeInMB() * 1024 * 1024;
        ICache kc;
        if (MemoryMeter.isInitialized())
        {
            kc = ConcurrentLinkedHashCache.create(keyCacheInMemoryCapacity);
        }
        else
        {
            ...
            EntryWeigher weigher = new EntryWeigher()
            {
                public int weightOf(KeyCacheKey key, RowIndexEntry entry)
                {
                    return key.key.length + entry.serializedSize();
                }
            };
            kc = ConcurrentLinkedHashCache.create(keyCacheInMemoryCapacity, weigher);
        }
        AutoSavingCache keyCache = new AutoSavingCache(kc, CacheType.KEY_CACHE, new KeyCacheSerializer());

        ...
        return keyCache;         
    }

[/cc]




# RowCache



[cc lang="java"]
// org.apache.cassandra.service.CacheService
public final AutoSavingCache rowCache;
[/cc]

RowCache caches entire row and removes all disk IO.

Like KeyCacheKey, _RowCacheKey_ keeps info to uniquely identify the colunm family the key belongs to. Unlike KeyCache, SSTable has no reference to RowCache. 

[cc lang="java"]
public class RowCacheKey implements CacheKey, Comparable
{
    public final UUID cfId;
    public final byte[] key;
    ...
}
[/cc]

RowCache has a configurable maximum size:

[cc lang="yaml"]
# Maximum size of the row cache in memory.
# Default value is 0, to disable row caching.
row_cache_size_in_mb: 0
[/cc]

RowCache is disabled by default not because its default size is 0 but **hard-coded**. 

[cc lang="java"]
// org.apache.cassandra.config

public final class CFMetaData
{
    ...
    public final static Caching DEFAULT_CACHING_STRATEGY = Caching.KEYS_ONLY;
    ...
    //OPTIONAL
    private volatile Caching caching = DEFAULT_CACHING_STRATEGY;
}
[/cc]

The DEFFAULT_CACHING_STRATEGY could be overwritten when creating a new column family.
[cc lang]
CREATE COLUMN FAMILY foobar
WITH caching = ALL / ROWS_ONLY
...
[/cc]

RowCache could choose between _ConcurrentLinkedHashCache_ and _SerializingCache_

[cc lang="yaml"]
# The provider for the row cache to use.
#
# Supported values are: ConcurrentLinkedHashCacheProvider, SerializingCacheProvider
#
# SerializingCacheProvider serialises the contents of the row and stores
# it in native memory, i.e., off the JVM Heap. Serialized rows take
# significantly less memory than "live" rows in the JVM, so you can cache
# more rows in a given memory footprint.  And storing the cache off-heap
# means you can use smaller heap sizes, reducing the impact of GC pauses.
#
# It is also valid to specify the fully-qualified class name to a class
# that implements org.apache.cassandra.cache.IRowCacheProvider.
#
# Defaults to SerializingCacheProvider
row_cache_provider: SerializingCacheProvider
[/cc]

The comments have introduced one of their differences: ConcurrentLinkedHashCache keeps data on heap while SerializingCache keeps data on stack (off-heap). There is more subtlety for the latter. Since SerializingCache serializes rows, deserializing a wide range of rows would take a lot of time, which could cancel the improved latency brought by RowCache (I have gone into cases where reading through SerializingCache with get_range_slice is even slower than reading SSTables while ConcurrentLinkedHashCache is not feasible either considering the memory pressure on heap). Another difference lies in _Caches at runtime_ which we'll talk about later.





# Design Concerns



The cache on Cassandra is integrated with the database. [Datastax documentation](http://www.datastax.com/dev/blog/caching-in-cassandra-1-1) explains the design concerns behind it. Let me elaborate on each point.



## Distributed by default




<blockquote>Cassandra takes care of distributing your data around the cluster for you. You don’t need to rely on an external library like [ketama](https://github.com/RJ/ketama) to spread your cache across multiple servers.</blockquote>


ketama ia a C library for [consist hashing](http://en.wikipedia.org/wiki/Consistent_hashing), which Cassandra has already implemented to distribute its data. Being local to data, per-node caches get distributed **by default**. 



## Architectural simplicity




<blockquote>Architectural simplicity: The less moving pieces in your architecture, the less combinations of things there are to go wrong. Troubleshooting a separate caching tier is something experienced ops teams prefer to avoid.</blockquote>


As said in _Distributed by default_.


## Cache coherence




<blockquote>When your cache is external to your database, your client must update both. Whether it updates the cache first, or the database, when you have server failure you will end up with data in the cache that is older (or newer) than what is in the database, indefinitely. With an integrated cache, cached data will always exactly match what’s in the database.</blockquote>



This [developer blog](http://www.datastax.com/dev/blog/maximizing-cache-benefit-with-cassandra) has a nice graph illustrating caches at runtime.



![cache](https://lh5.googleusercontent.com/-0CIStGalh10/UDwbnDrUrVI/AAAAAAAAAmA/3frv4-3onzQ/w362-h321-n-k/cache_hits.png)



KeyCache is updated by the reader (refer to  [cci]org.apache.cassandra.io.sstable.SSTableReader.getPosition[/cci]), which is omitted by the graph. We mentioned that KeyCache is per-SSTable and thus is written only once since SSTable is immutable. Hence, there is no chance to leave KeyCache with stale data.

Like KeyCache, the graph shows that RowCache is updated by the reader as well. The subtlety here is that RowCache is not always updated. When reading with _get_ and _get_slice_, the underlying method is  [cci]ColumnFamilyStore.getThroughCache[/cci] which does what the graph depicts; when reading with _get_range_slice_ it's [cci]ColumnFamilyStore.getRawCachedRow[/cci] which will not cache the row if not present. As for the reasons, refer to [CASSANDRA-1302](https://issues.apache.org/jira/browse/CASSANDRA-1302).

This has yet been the whole story, however. Note that if we update a column in Memtable the stale data in RowCache won't be updated. When reading that column we would always get the stale data since RowCache is a map and the key does exist in the map. Hence, to preserve cache coherence a writer either invalidates (with SerializingCache) or updates-in-place (with ConcurrentLinkedHashCache).

Let's do an experiment to verify it.





  1. Start Cassandra and _cassandra-cli_



  2. 
Create a new column family _test_ (we use an existent keyspace _tpch_).
[cc]
[default@unknown] use tpch;
[default@tpch] create column family test 
with key_validation_class = UTF8Type 
and comparator = UTF8Type 
and caching = all;
[/cc]




  3. 
We add a name 'william' and read thrice; then update it with its short form 'bill' and read thrice again.
(we use an arbitrary key here)
[cc highlight="17,18,19"]
[default@tpch] assume test validator as utf8;
[default@tpch] set test[3][name] = 'william';
Value inserted.
Elapsed time: 7.03 msec(s).
[default@tpch] get test[3][name];
=> (column=name, value=william, timestamp=1353382334047000)
Elapsed time: 32 msec(s).
[default@tpch] get test[3][name];
=> (column=name, value=william, timestamp=1353382334047000)
Elapsed time: 7.04 msec(s).
[default@tpch] get test[3][name];
=> (column=name, value=william, timestamp=1353382334047000)
Elapsed time: 6.07 msec(s).
[default@tpch] set test[3][name] = 'bill';
Value inserted.
Elapsed time: 5.39 msec(s).
[default@tpch] get test[3][name];
=> (column=name, value=bill, timestamp=1353382418135000)
Elapsed time: 8.36 msec(s).
[default@tpch] get test[3][name];
=> (column=name, value=bill, timestamp=1353382418135000)
Elapsed time: 5.76 msec(s).
[default@tpch] get test[3][name];
=> (column=name, value=bill, timestamp=1353382418135000)
Elapsed time: 7.43 msec(s).[/cc]



We read thrice to detect whether the row is already in RowCache. If not, we may have significant improved latency on second read and the latency stabilizes afterwards. Otherwise, we see no significant latency differences over the three reads.

Repeat it with SerializingCache (after invalidateRowCache through jconsole)

[cc highlight="16,17,18"]
[default@tpch] set test[3][name] = 'william';
Value inserted.
Elapsed time: 4.03 msec(s).
[default@tpch] get test[3][name];
=> (column=name, value=william, timestamp=1353384086002000)
Elapsed time: 22 msec(s).
[default@tpch] get test[3][name];
=> (column=name, value=william, timestamp=1353384086002000)
Elapsed time: 7.81 msec(s).
[default@tpch] get test[3][name];
=> (column=name, value=william, timestamp=1353384086002000)
Elapsed time: 6.26 msec(s).
[default@tpch] set test[3][name] = 'bill';
Value inserted.
Elapsed time: 5.25 msec(s).
[default@tpch] get test[3][name];
=> (column=name, value=bill, timestamp=1353384126137000)
Elapsed time: 12 msec(s).
[default@tpch] get test[3][name];
=> (column=name, value=bill, timestamp=1353384126137000)
Elapsed time: 5.86 msec(s).
[default@tpch] get test[3][name];
=> (column=name, value=bill, timestamp=1353384126137000)
Elapsed time: 6.24 msec(s).
[/cc]

Only one thing is different (compare the highlighted lines), we see a latency jitter after updating. This proves that with SerializingCache writers invalidate cache (updates would involve deserialize, merge and reserialize, which is slow. [CASSANDRA-1969](https://issues.apache.org/jira/browse/CASSANDRA-1969) covers more details) while with ConcurrentLinkedHashCache writers update cache in place. 

The underlying method is [cci]ColumnFamilyStore.updateRowCache[/cci].



## RowCacheSentinel



Nonetheless, RowCache could still have such issue as dirty write which breaks cache coherence since read-and-cache is not an atomic operation. Consider the above example again (with ConcurrentLinkedHashCache):



    1. [cci]set test[3][name] = 'william'; [/cci] as before


    2. We invoke [cci]get test[3][name];[/cci] for the first time


    3. Data are not in the RowCache and fetched from Memtable/SSTable but yet cached


    4. [cci]set test[3][name] = 'bill';[/cci] and writer updates RowCache as well


    5. Cache data and overwrite 'bill' with 'william'



The second write is lost (with SerializingCache, there's no worry over dirty write but cache coherence is not preserved since reader caches stale data into cache anyway).

To solve the problem, Cassandra uses an _RowCacheSentinel_ to mark reader's state before read-and-cache. When seeing the sentinel, reader does a normal read without cache and remove the sentinel. If writer sees the sentinel and invalidates it right before reader caches (after line 23 and before line 31 below), reader will leave the cache empty. Hence, we avoid dirty write and cache coherence is preserved.

[cc lang="java"]
// org.apache.cassandra.db.ColumnFamilyStore
    private ColumnFamily getThroughCache(UUID cfId, QueryFilter filter)
    {
        assert isRowCacheEnabled()
               : String.format("Row cache is not enabled on column family [" + getColumnFamilyName() + "]");

        RowCacheKey key = new RowCacheKey(cfId, filter.key);

        // attempt a sentinel-read-cache sequence.  if a write invalidates our sentinel, we'll return our
        // (now potentially obsolete) data, but won't cache it. see CASSANDRA-3862
        IRowCacheEntry cached = CacheService.instance.rowCache.get(key);
        if (cached != null)
        {
            if (cached instanceof RowCacheSentinel)
            {
                // Some other read is trying to cache the value, just do a normal non-caching read
                return getTopLevelColumns(filter, Integer.MIN_VALUE, false);
            }
            return (ColumnFamily) cached;
        }

        RowCacheSentinel sentinel = new RowCacheSentinel();
        boolean sentinelSuccess = CacheService.instance.rowCache.putIfAbsent(key, sentinel);

        try
        {
            ColumnFamily data = getTopLevelColumns(QueryFilter.getIdentityFilter(filter.key, new QueryPath(columnFamily)),
                                                   Integer.MIN_VALUE,
                                                   true);
            if (sentinelSuccess && data != null)
                CacheService.instance.rowCache.replace(key, sentinel, data);

            return data;
        }
        finally
        {
            if (sentinelSuccess && data == null)
                CacheService.instance.rowCache.remove(key);
        }
    }

[/cc]

Overall, with both KeyCache and RowCache, we make sure no stale data could sneak into them.




## Redundancy




<blockquote>
If you have temporarily lose a memcached server, your database will receive the read load from that set of keys until that server comes back online. With Cassandra’s fully distributed cache, your client can read from another (cached) replica of the data instead, reducing the impact on your cluster.
</blockquote>


Since cache is local to data, it is replicated as well.



## Solving the cold start problem




<blockquote>
The cold start problem refers to the scenario where after an outage such as a power failure, where all your caches have restarted and are thus empty (“cold”), your database will be hit with the entire application read load, including a storm of retries as initial requests time out. Usually the reason you have memcached in the first place is that your database can’t handle this entire read load, so your ops team is going to be sweating bullets trying to throttle things until they get the caches reheated. Since Cassandra provides a durable database behind the cache, it can save your cache to disk periodically and read the contents back in when it restarts, so you never have to start with a cold cache.
</blockquote>



In Cassandra, caches are persisted to disk periodically. The physical position is configurable in

[cc lang="yaml"]
# saved caches
saved_caches_directory: /home/manuzhang/cassandra/saved_caches
[/cc]

A list of my saved caches:

[cc]
tpch-customer-KeyCache-b.db
tpch-customer-RowCache-b.db
tpch-orders-KeyCache-b.db
tpch-orders-RowCache-b.db
[/cc]

Further, we could configure the period and number of keys to save
[cc lang="yaml"]
# Duration in seconds after which Cassandra should save the keys cache.
# Default is 14400 or 4 hours.
key_cache_save_period: 14400

# Number of keys from the key cache to save
# Disabled by default, meaning all keys are going to be saved
# key_cache_keys_to_save: 100

# Duration in seconds after which Cassandra should
# safe the row cache. Caches are saved to saved_caches_directory as specified
# in this configuration file.
#
# Saved caches greatly improve cold-start speeds, and is relatively cheap in
# terms of I/O for the key cache. Row cache saving is much more expensive and
# has limited use.
#
# Default is 0 to disable saving the row cache.
row_cache_save_period: 0

# Number of keys from the row cache to save
# Disabled by default, meaning all keys are going to be saved
# row_cache_keys_to_save: 100
[/cc]
Note that only RowCacheKey is saved for row cache saving. On start up, Cassandra will rebuild RowCache from SSTables (refer to _org.apache.cassandra.service.CacheService.RowCacheSerializer_). The rebuilding could cost hours of time (I have even run into an OOM). Hence, saving RowCache is disabled by default (this is not hard-coded).  



# Take away


That's it. It's a long post so to check out whether we have learned more about caches in Cassandra, we may ask ourselves the following questions:

 
    1. What are KeyCache and RowCache?

 
    2. How are they distributed in a Cassandra cluster?

 
    3. How do they work at runtime (How is cache coherence preserved)?

 
    4. Are they durable?


