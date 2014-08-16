---
author: manuzhang
comments: true
date: 2012-08-22 16:01:10+00:00
layout: post
published: false
slug: cassandra-codebase-cassandra-daemon
title: Cassandra codebase -- Cassandra Daemon
wordpress_id: 297
categories:
- database
- 实习笔记
tags:
- Cassandra
---

I've been exploring into Cassandra codebase for a while. I found two nice guides [here](http://www.slideshare.net/gdusbabek/getting-to-know-the-cassandra-codebase) and [here](http://www.slideshare.net/gdusbabek/cassandra-codebase-2011) by Gary Dusbabek. There could be more and those slides were one year ago (my Cassandra version now is **1.2.0-SNAPSHOT**) but that's enough to get started and follow up. In my illustration, I'll quote the comments as much as possible.



I [run Cassandra in Eclipse](http://wiki.apache.org/cassandra/RunningCassandraInEclipse). Here are my VM arguments and they will be handy later.




    
    <code>-Dcassandra.config=file:/home/manuzhang/git/cassandra-trunk/conf/cassandra.yaml
    -Dcassandra-foreground
    -ea -Xmx1G
    -Dlog4j.configuration=file:/home/manuzhang/git/cassandra-trunk/conf/log4j-server.properties
    </code>



Let's get our feet wet with the startup process. It's in `org.apache.cassandra.service.CassandraDaemon`.




    
    <code>/* org.apache.cassandra.service.CassandraDaemon */
    private static final CassandraDaemon instance = new CassandraDaemon();
    
    public static void main(String[] args)
    {
        instance.activate();
    }
    </code>



<!-- more -->

The CassandraDaemon is an abstraction for a Cassandra daemon service, which defines not only a way to activate and deactivate it, but also hooks into its lifecycle methods (setup, start, stop).




    
    <code>/* org.apache.cassandra.service.CassandraDaemon */
    public void activate()
    {
        String pidFile = System.getProperty("cassandra-pidfile");
    
        try
        {
            setup();
    
            if (pidFile != null)
            {
                new File(pidFile).deleteOnExit();
            }
    
            if (System.getProperty("cassandra-foreground") == null)
            {
                System.out.close();
                System.err.close();
            }
    
            start();
        }
        catch (Throwable e)
        {
            logger.error("Exception encountered during startup", e);
    
            // try to warn user on stdout too, if we haven't already detached
            e.printStackTrace();
            System.out.println("Exception encountered during startup: " + e.getMessage());
    
            System.exit(3);
        }
    }
    </code>



The cassandra-pidfile is where the process ID is written to (useful for shutdown via `kill $(cat pidfile)`). I didn't set it in my VM arguments so `pidfile == null`. But I did set the cassandra-foreground, and thus stdout and stderr won't get detached.



Next, we'll dive into two important methods, `setup` and `start`.





# I. org.apache.cassandra.service.CassandraDaemon.setup()



It mainly does the following things:







  * Reads configuration: DatabaseDescriptor.loadyaml()


  * Initializes CacheService


  * Loads schema: DatabaseDescriptor.loadSchemas()


  * Scrub directories (two steps)


  * Initializes storage (keyspaces + column families)


  * Initializes GCInspector


  * Commit log recovery: CommitLog.recover()


  * StorageService.initServer()


  * Initializes ThriftServer and transport server





## Reads configuration



There is no explicit call for `loadYaml` in setup and all I find is:




    
    <code>/* org.apache.cassandra.service.CassandraDaemon.setup() */
    // check all directories(data, commitlog, saved cache) for existence and permission
    Iterable<String> dirs = Iterables.concat(Arrays.asList(DatabaseDescriptor.getAllDataFileLocations()),
                                             Arrays.asList(new String[] {DatabaseDescriptor.getCommitLogLocation(), DatabaseDescriptor.getSavedCachesLocation()}));
    </code>



It's impossible to check the directories whose locations are unknown before reading the configuration file. My best guess is the loadYaml method is invoked in a static clause in the `DatabaseDescriptor` class and it is. The first reference to the static method of DatabaseDescriptor causes it to be initialized and thus the static clause is executed.




    
    <code>/* org.apache.cassandra.config.DatabaseDescriptor */
    static
    {
        if (Config.getLoadYaml())
            loadYaml();
        else
            conf = new Config();
    }
    </code>



I prefer not to list all the configuration options in this post, but return to them when needed.



Remember I set **cassandra.config** earlier? It's now fetched:




    
    <code>/* org.apache.cassandra.config.DatabaseDescriptor */
    
    static void loadYaml()
    {
        ...
        URL url = getStorageConfigURL();
        ...     
    }
    
    static URL getStorageConfigURL() throws ConfigurationException
    {
        String configUrl = System.getProperty("cassandra.config");
        ...
    }
    </code>



Thus, the configuration file **cassandra.yaml** can be located in `cassandra-trunk/conf`, and it's for us to open up and tune our Cassandra. Get help [here](http://wiki.apache.org/cassandra/StorageConfiguration) and the yaml file is also well commented.



While loading configurations, we hardcode metadata for SystemKeyspace and store it into the **global schema (org.apache.cassandra.config.Schema)**. Note that the SystemKeyspace has yet existed, not until **Scrubs directories**.




    
    <code>/* org.apache.cassandra.config.DatabaseDescriptor */
    static void loadYaml()
    {
        ...     
        // Hardcoded system tables
        KSMetaData systemMeta = KSMetaData.systemKeyspace();
        for (CFMetaData cfm : systemMeta.cfMetaData().values())
           Schema.instance.load(cfm);
    
        Schema.instance.addSystemTable(systemMeta);
        ...
    }
    </code>



SystemKeyspace keeps 12 ColumnFamilies storing metadata for the local node, as well as hinted handoff information. We'll learn about those CFs when using them.




    
    <code>/* org.apache.cassandra.config.KSMetaData */
    public static KSMetaData systemKeyspace()
    {
        List<CFMetaData> cfDefs = Arrays.asList(CFMetaData.LocalCf,
                                                CFMetaData.PeersCf,
                                                CFMetaData.HintsCf,
                                                CFMetaData.IndexCf,
                                                CFMetaData.NodeIdCf,
                                                CFMetaData.SchemaKeyspacesCf,
                                                CFMetaData.SchemaColumnFamiliesCf,
                                                CFMetaData.SchemaColumnsCf,
                                                CFMetaData.OldStatusCf,
                                                CFMetaData.OldHintsCf,
                                                CFMetaData.MigrationsCf,
                                                CFMetaData.SchemaCf);
        return new KSMetaData(Table.SYSTEM_TABLE, LocalStrategy.class, Collections.<String, String>emptyMap(), true, cfDefs);
    }
    </code>



The in-memory global schema is to speed up lookups for Table (Keyspace) and ColumnFamily.




    
    <code>/* org.apache.cassandra.config.Schema */
    
    /* metadata map for faster table lookup */
    private final Map<String, KSMetaData> tables = new NonBlockingHashMap<String, KSMetaData>();
    
    /* Table objects, one per keyspace. Only one instance should ever exist for any given keyspace. */
    private final Map<String, Table> tableInstances = new NonBlockingHashMap<String, Table>();
    
    /* metadata map for faster ColumnFamily lookup */
    private final BiMap<Pair<String, String>, UUID> cfIdMap = HashBiMap.create();
    
    // mapping from old ColumnFamily Id (Integer) to a new version which is UUID
    private final BiMap<Integer, UUID> oldCfIdMap = HashBiMap.create();
    </code>



When hardcoding SystemKeyspace metadata, we store its ColumnFamilies info into **cfIdMap**, its own info into **tables**. Since we have a particular class `org.apache.cassandra.db.SystemTable` for SystemKeyspace, its instance is not stored in **tableInstances**.





## Initializes CacheService




    
    <code>if (CacheService.instance == null) // should never happen
        throw new RuntimeException("Failed to initialize Cache Service.");
    </code>





## Scrubs directories and Loads schema



Scrubing directories is performed in two steps, separated by loading schema. In the first step, we scrub SystemKeyspace directories. Actually, we have to initialize SystemKeyspace (create the directories) for the first run. Only after that are we able to load metadata of other keyspaces, which are stored in SystemKeyspace. Finally, we move on to scrub directories of the rest keyspaces in the second step.




    
    <code>/* org.apache.cassandra.service.CassandraDaemon.setup() */
    for (CFMetaData cfm : Schema.instance.getTableMetaData(Table.SYSTEM_TABLE).values())
        ColumnFamilyStore.scrubDataDirectories(Table.SYSTEM_TABLE, cfm.cfName);
    ...
    // load keyspace descriptions
    DatabaseDescriptor.loadSchemas();
    ...
    
    // clean up debris in the rest of the tables
    for (String table : Schema.instance.getTables())
    {
        for (CFMetaData cfm : Schema.instance.getTableMetaData(table).values())
        {
            ColumnFamilyStore.scrubDataDirectories(table, cfm.cfName);
        }
    }
    </code>



As in the case of **DatabaseDescriptor**, a static clause is executed on initialization of **Table**. We can see that all directories (data_file_directory, commitlog_directory, saved_caches_directory) are created (existent directories won't be created again).




    
    <code>/* org.apache.cassandra.db.Table */
    static
    {
        if (!StorageService.instance.isClientMode())
        {
            try
            {
                DatabaseDescriptor.createAllDirectories();
            }
            catch (IOException ex)
            {
                throw new IOError(ex);
            }
        }
    }
    </code>



In **loadSchemas** we load metadata of the rest keyspaces into the **global schema**.




    
    <code>/* org.apache.cassandra.config.DatabaseDescriptor */
    public static void loadSchemas() throws IOException
    {
        ...        
        Schema.instance.load(DefsTable.loadFromTable());
        ...
    }
    </code>



This is the first time we make use of ColumnFamily in SystemKeyspace. SCHEMA_{KEYSPACES, COLUMNFAMILIES, COLUMNS}_CF are used to store Keyspace/ColumnFamily attributes to make schema load/distribution easy.




    
    <code>/* org.apache.cassandra.db.DefsTable */
    public static Collection<KSMetaData> loadFromTable() throws IOException
    {
        List<Row> serializedSchema = SystemTable.serializedSchema(SystemTable.SCHEMA_KEYSPACES_CF);
    
        List<KSMetaData> keyspaces = new ArrayList<KSMetaData>(serializedSchema.size());
    
        for (Row row : serializedSchema)
        {
            if (row.cf == null || (row.cf.isMarkedForDelete() && row.cf.isEmpty()))
                continue;
    
            keyspaces.add(KSMetaData.fromSchema(row, serializedColumnFamilies(row.key)));
        }
    
        return keyspaces;
    }
    
    /* org.apache.cassandra.config.Schema */
    public Schema load(Collection<KSMetaData> tableDefs)
    {
        for (KSMetaData def : tableDefs)
            load(def);
    
        return this;
    }
    
    /* org.apache.cassandra.config.Schema */
    public Schema load(KSMetaData keyspaceDef)
    {
        for (CFMetaData cfm : keyspaceDef.cfMetaData().values())
            load(cfm);
    
        setTableDefinition(keyspaceDef);
    
        return this;
    }
    
    /* org.apache.cassandra.config.Schema */
    public void setTableDefinition(KSMetaData ksm)
    {
        if (ksm != null)
            tables.put(ksm.name, ksm);
    }
    </code>



Now, Cassandra is able to "clean up debris in the rest of the tables" and initialize the rest tables (keyspaces) as well.





## Initializes Storage



To initialize the keyspaces (excluding SystemKeyspace), we draw out what we've just deposited from the **global schema**.




    
    <code>/* org.apache.cassandra.service.CassandraDaemon.setup() */
    for (String table : Schema.instance.getTables())
    {
        if (logger.isDebugEnabled())
            logger.debug("opening keyspace " + table);
        Table.open(table);
    }
    
    /* org.apache.cassandra.db.Table */ 
    public static Table open(String table)
    {
        return open(table, Schema.instance, true);
    }
    
    /* org.apache.cassandra.db.Table */ 
    private static Table open(String table, Schema schema, boolean loadSSTables)
    {
        Table tableInstance = schema.getTableInstance(table);
    
        if (tableInstance == null)
        {
            ...
        }
        return tableInstance;
    }
    </code>



Since we've already loaded keyspace metadata in the previous step, `Table.open(table)` does nothing here.





## Initializes GCInspector



When thresholds are reached, Cassandra periodically flushes in-memory data structures (memtables) to SSTable

data files for persistent storage of column family data. GCInspector is responsible for inspecting the memory usage and it is run every second by default.




    
    <code>/* org.apache.cassandra.service.CassandraDaemon.setup() */
    GCInspector.instance.start();
    </code>





## Commit log recovery




    
    <code>/* org.apache.cassandra.service.CassandraDaemon.setup() */
    CommitLog.instance.recover();
    </code>





## StorageService.initServer()




    
    <code>/* org.apache.cassandra.service.CassandraDaemon.setup() */
    StorageService.instance.initServer();
    
    /* org.apache.cassandra.service.StorageService */
    public synchronized void initServer() throws IOException, ConfigurationException
    {
        initServer(RING_DELAY);
    }
    </code>



**RING_DELAY** is the delay after which we assume ring has stabilized. Its default value is 30000 if not specified in VM arguments.




    
    <code>/* org.apache.cassandra.service.StorageService.initServer */
    if (Boolean.parseBoolean(System.getProperty("cassandra.join_ring", "true")))
    {
        joinTokenRing(delay);
    }
    else
    {
        logger.info("Not joining ring as requested. Use JMX (StorageService->joinRing()) to initiate ring joining");
    }
    </code>



We do several important things here.





### start gossip service




    
    <code>/* org.apache.cassandra.service.StorageService.joinTokenRing */
    Gossiper.instance.register(this);
    Gossiper.instance.register(migrationManager);
    Gossiper.instance.start(SystemTable.incrementAndGetGeneration());
    </code>



Note that we have to start the gossip service before we see info on other nodes.





### start messaging service




    
    <code>/* org.apache.cassandra.service.StorageService.joinTokenRing */
    MessagingService.instance().listen(FBUtilities.getLocalAddress());
    </code>





### start hintedhandoff manager




    
    <code>/* org.apache.cassandra.service.StorageService.joinTokenRing */
    HintedHandOffManager.instance.start();
    </code>





### bootstrap



Adding new nodes is called "bootstrapping."



Look for more info on bootstrap [here](http://wiki.apache.org/cassandra/Operations#Bootstrap). **AutoBootstrap** is hardcoded **on** so there is no need to add it to the configuration file.




    
    <code>/* org.apache.cassandra.config.Config */
    public Boolean auto_bootstrap = true;
    </code>



We bootstrap if we haven't successfully bootstrapped before, as long as we are not a seed.




    
    <code>/* org.apache.cassandra.service.StorageService.joinTokenRing */
    if (DatabaseDescriptor.isAutoBootstrap()
            && !SystemTable.bootstrapComplete()
            && !DatabaseDescriptor.getSeeds().contains(FBUtilities.getBroadcastAddress()))
    {
        ...
    }
    else
    {
        ...
    }
    </code>





## Initializes ThriftServer and transport server




    
    <code>/* org.apache.cassandra.service.CassandraDaemon.setup() */
    // Thift
    InetAddress rpcAddr = DatabaseDescriptor.getRpcAddress();
    int rpcPort = DatabaseDescriptor.getRpcPort();
    thriftServer = new ThriftServer(rpcAddr, rpcPort);
    
    // Native transport
    InetAddress nativeAddr = DatabaseDescriptor.getNativeTransportAddress();
    int nativePort = DatabaseDescriptor.getNativeTransportPort();
    nativeServer = new org.apache.cassandra.transport.Server(nativeAddr, nativePort);
    </code>



According to the default setting in `conf/cassandra.yaml`, Native transport server won't be started though.




    
    <code># Whether to start the native transport server.
    # Currently, only the thrift server is started by default because the native
    # transport is considered beta.
    start_native_transport: false
    </code>



This is also verified by its corresponding value in `org.apache.cassandra.config.Config`:




    
    <code>public Boolean start_native_transport = false;
    </code>





# II. org.apache.cassandra.service.CassandraDaemon.start()





## start CassandraServer




    
    <code>/* org.apache.cassandra.service.CassandraDaemon */
    public void start()
    {
        String rpcFlag = System.getProperty("cassandra.start_rpc");
        if ((rpcFlag != null && Boolean.parseBoolean(rpcFlag)) || (rpcFlag == null && DatabaseDescriptor.startRpc()))
            thriftServer.start();
        else
            logger.info("Not starting RPC server as requested. Use JMX (StorageService->startRPCServer()) to start it");
    
        String nativeFlag = System.getProperty("cassandra.start_native_transport");
        if ((nativeFlag != null && Boolean.parseBoolean(nativeFlag)) || (nativeFlag == null && DatabaseDescriptor.startNativeTransport()))
            nativeServer.start();
        else    /* org.apache.cassandra.service.CassandraDaemon.start() */
            logger.info("Not starting native transport as requested. Use JMX (StorageService->startNativeTransport()) to start it");
    }
    </code>



Since we only start thriftServer by default, we will ignore `nativeServer.start()` for the current.




    
    <code>/* org.apache.cassandra.thrift.ThriftServer */
    public void start()
    {
        if (server == null)
        {
            server = new ThriftServerThread(address, port);
            server.start();
        }
    }
    </code>



The _ThriftServerThread_ establishes the RPC server against [Apache Thrift](http://en.wikipedia.org/wiki/Apache_Thrift) framework. Our CassandraServer is seated right at the processor of the Thrift Server.




    
    <code>/* org.apache.cassandra.thrift.ThriftServer */   
    
    // Simple class to run the thrift connection accepting code in separate thread of control.   
    private static class ThriftServerThread extends Thread  
    {  
            private TServer serverEngine;
    
            public ThriftServerThread(InetAddress listenAddr, int listenPort)
            {
               // now we start listening for clients
               final CassandraServer cassandraServer = new CassandraServer();
               Cassandra.Processor processor = new Cassandra.Processor(cassandraServer);
    
               ...
    
               if (DatabaseDescriptor.getRpcServerType().equalsIgnoreCase(SYNC))
               {
                   // ThreadPool Server and will be invocation per connection basis.
                   ...
                   ExecutorService executorService = new CleaningThreadPool(cassandraServer.clientState, serverArgs.minWorkerThreads, serverArgs.maxWorkerThreads);
                   serverEngine = new CustomTThreadPoolServer(serverArgs, executorService);    
               }
               else
               {
                   if (DatabaseDescriptor.getRpcServerType().equalsIgnoreCase(ASYNC))
                   {
                       // This is single threaded hence the invocation will be all in one thread.
                       ...
                       serverEngine = new CustomTNonBlockingServer(serverArgs);
    
                   }  
                   else if (DatabaseDescriptor.getRpcServerType().equalsIgnoreCase(HSHA))
                   {
                       // This is NIO selector service but the invocation will be Multi-Threaded with the Executor service.
                       ...
                       ExecutorService executorService = new JMXEnabledThreadPoolExecutor(DatabaseDescriptor.getRpcMinThreads(),
                                                                                           DatabaseDescriptor.getRpcMaxThreads(),
                                                                                           60L,
                                                                                           TimeUnit.SECONDS,
                                                                                           new SynchronousQueue<Runnable>(),
                                                                                           new NamedThreadFactory("RPC-Thread"), "RPC-THREAD-POOL");
                       // Check for available processors in the system which will be equal to the IO Threads.
                       serverEngine = new CustomTHsHaServer(serverArgs, executorService, Runtime.getRuntime().availableProcessors());
    
                   }
               }
            }
    
            public void run()
            {
                logger.info("Listening for thrift clients...");
                serverEngine.serve();
            }
    
            public void stopServer()
            {
                logger.info("Stop listening to thrift clients");
                serverEngine.stop();
            }
        }    
    </code>



What is **rpc_server_type**? Look it up in **conf/cassandra.yaml**:




    
    <code># Cassandra provides three options for the RPC Server:
    #
    # sync  -> One thread per thrift connection. For a very large number of clients, memory
    #          will be your limiting factor. On a 64 bit JVM, 128KB is the minimum stack size
    #          per thread, and that will correspond to your use of virtual memory (but physical memory
    #          may be limited depending on use of stack space).
    #
    # hsha  -> Stands for "half synchronous, half asynchronous." All thrift clients are handled
    #          asynchronously using a small number of threads that does not vary with the amount
    #          of thrift clients (and thus scales well to many clients). The rpc requests are still
    #          synchronous (one thread per active request).
    #
    # The default is sync because on Windows hsha is about 30% slower.  On Linux,
    # sync/hsha performance is about the same, with hsha of course using less memory.
    rpc_server_type: sync
    </code>



Although the **async** option is left out, the comment "This is single threaded hence the invocation will be all in one thread." has explained itself (also verified by the fact that no _ExecutorService_ is needed). Actually, the underlying mechanism is [Selector](http://tutorials.jenkov.com/java-nio/selectors.html), which enables a single thread to handle multiple [ServerSocketChannel](http://tutorials.jenkov.com/java-nio/server-socket-channel.html)s. And **hsha** is a pool of such threads that equipped with **Selector**.





# In Summary



That's pretty much and I know I have yet covered all the bolts and nuts. Some topics such as CommitLog recovery and bootstrapping merit detailed introduction in a separate post. I've tried my best to keep it concise and not to leave any crux out. Hope I've done enough to get you on board. Now, we are setting out!



