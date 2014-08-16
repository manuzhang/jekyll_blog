---
author: manuzhang
comments: true
date: 2012-09-01 06:54:20+00:00
layout: post
published: false
slug: cassandra-codebase-create-keyspace-column-family
title: Cassandra codebase -- Create Keyspace && Column Family
wordpress_id: 492
categories:
- database
- 实习笔记
tags:
- Cassandra
---

In this post, we are going to explore the issue of creating keyspace and column family. Bear in mind that when a new keyspace or column family is created at a node, it gets propagated to all the live nodes.





# What happens when we create a new keyspace ?



So what happens when we execute the following command in [CassandraCli](http://wiki.apache.org/cassandra/CassandraCli):




    
    <code>[default@unknown] create keyspace Twissandra;
    </code>



Of course we create a new keyspace named Twissandra, I mean the process behind the scenes, which we will dive into in a minute. We will do the same thing for creating a new column family.



<!-- more -->

So when it received the request, Thrift Server handed the keyspace definition packaged in a `KsDef` object to its processor, CassandraServer, which will invoke the corresponding method, `system_add_keyspace`.




    
    <code>/* org.apache.cassandra.service.CassandraServer */
    public String system_add_keyspace(KsDef ks_def)
    throws InvalidRequestException, SchemaDisagreementException, TException
    {
        ... // validation
    
        try
        {
            Collection<CFMetaData> cfDefs = new ArrayList<CFMetaData>(ks_def.cf_defs.size());
            for (CfDef cf_def : ks_def.cf_defs)
            {
                cf_def.unsetId(); // explicitly ignore any id set by client (same as system_add_column_family)
                CFMetaData cfm = CFMetaData.fromThrift(cf_def);
                cfm.addDefaultIndexNames();
                cfDefs.add(cfm);
            }
            // create replica placement strategy here
            MigrationManager.announceNewKeyspace(KSMetaData.fromThrift(ks_def, cfDefs.toArray(new CFMetaData[cfDefs.size()])));
            return Schema.instance.getVersion().toString();
        }
        catch (ConfigurationException e)
        {
            InvalidRequestException ex = new InvalidRequestException(e.getMessage());
            ex.initCause(e);
            throw ex;
        }
    }
    </code>



KsDef and CfDef are structures at thrift interface level (do not confuse [Cassandra Thrift API](http://wiki.apache.org/cassandra/API) with [Apache Thrift](http://thrift.apache.org/)), and they have storage level counterparts KSMetaData and CFMetaData respectively. `toThrift` and `fromThrift` methods are meant to make conversions between them.



Once getting this KSMetaData (with CFMetaData contained), we want to spread the news to all the live endpoints. Thus, the work is dispatched to MigrationManager.




    
    <code>/* org.apache.cassandra.service.MigrationManager */
    public static void announceNewKeyspace(KSMetaData ksm) throws ConfigurationException
    {
        // create replica placement strategy here 
        ksm.validate();
    
        if (Schema.instance.getTableDefinition(ksm.name) != null)
            throw new ConfigurationException(String.format("Cannot add already existing keyspace '%s'.", ksm.name));
    
        logger.info(String.format("Create new Keyspace: %s", ksm));
        announce(ksm.toSchema(FBUtilities.timestampMicros()));
    }
    </code>



Locally, we need to update schema information of three levels, Keyspace, ColumnFamily and Column, which is maintained as column families in SystemKeyspace. Check out the comments at the head of `org.apache.cassandra.db.DefsTable`




    
    <code>/**
     * SCHEMA_{KEYSPACES, COLUMNFAMILIES, COLUMNS}_CF are used to store Keyspace/ColumnFamily attributes to make schema
     * load/distribution easy, it replaces old mechanism when local migrations where serialized, stored in system.Migrations
     * and used for schema distribution.
     *
     * SCHEMA_KEYSPACES_CF layout:
     *
     * <key (AsciiType)>
     *   ascii => json_serialized_value
     *   ...
     * </key>
     *
     * Where <key> is a name of keyspace e.g. "ks".
     *
     * SCHEMA_COLUMNFAMILIES_CF layout:
     *
     * <key (AsciiType)>
     *     composite(ascii, ascii) => json_serialized_value
     * </key>
     *
     * Where <key> is a name of keyspace e.g. "ks"., first component of the column name is name of the ColumnFamily, last
     * component is the name of the ColumnFamily attribute.
     *
     * SCHEMA_COLUMNS_CF layout:
     *
     * <key (AsciiType)>
     *     composite(ascii, ascii, ascii) => json_serialized value
     * </key>
     *
     * Where <key> is a name of keyspace e.g. "ks".
     *
     * Column names where made composite to support 3-level nesting which represents following structure:
     * "ColumnFamily name":"column name":"column attribute" => "value"
     *
     * Example of schema (using CLI):
     *
     * schema_keyspaces
     * ----------------
     * RowKey: ks
     *  => (column=durable_writes, value=true, timestamp=1327061028312185000)
     *  => (column=name, value="ks", timestamp=1327061028312185000)
     *  => (column=replication_factor, value=0, timestamp=1327061028312185000)
     *  => (column=strategy_class, value="org.apache.cassandra.locator.NetworkTopologyStrategy", timestamp=1327061028312185000)
     *  => (column=strategy_options, value={"datacenter1":"1"}, timestamp=1327061028312185000)
     *
     * schema_columnfamilies
     * ---------------------
     * RowKey: ks
     *  => (column=cf:bloom_filter_fp_chance, value=0.0, timestamp=1327061105833119000)
     *  => (column=cf:caching, value="NONE", timestamp=1327061105833119000)
     *  => (column=cf:column_type, value="Standard", timestamp=1327061105833119000)
     *  => (column=cf:comment, value="ColumnFamily", timestamp=1327061105833119000)
     *  => (column=cf:default_validation_class, value="org.apache.cassandra.db.marshal.BytesType", timestamp=1327061105833119000)
     *  => (column=cf:gc_grace_seconds, value=864000, timestamp=1327061105833119000)
     *  => (column=cf:id, value=1000, timestamp=1327061105833119000)
     *  => (column=cf:key_alias, value="S0VZ", timestamp=1327061105833119000)
     *  ... part of the output omitted.
     *
     * schema_columns
     * --------------
     * RowKey: ks
     *  => (column=cf:c:index_name, value=null, timestamp=1327061105833119000)
     *  => (column=cf:c:index_options, value=null, timestamp=1327061105833119000)
     *  => (column=cf:c:index_type, value=null, timestamp=1327061105833119000)
     *  => (column=cf:c:name, value="aGVsbG8=", timestamp=1327061105833119000)
     *  => (column=cf:c:validation_class, value="org.apache.cassandra.db.marshal.AsciiType", timestamp=1327061105833119000)
     */
    </code>



The three-level updates are performed via RowMutation with the keyspace name (Twissandra here) as the key.



Firstly, `SystemTable.SCHEMA_KEYSPACES_CF`. Another thing we learn is that `durableWrites`, `strategyClass` and `strategyOptions` are all keyspace level options.




    
    <code>/* org.apache.cassandra.config.KSMetaData */
    public RowMutation toSchema(long timestamp)
    {
        RowMutation rm = new RowMutation(Table.SYSTEM_TABLE, SystemTable.getSchemaKSKey(name));
        ColumnFamily cf = rm.addOrGet(SystemTable.SCHEMA_KEYSPACES_CF);
    
        cf.addColumn(Column.create(durableWrites, timestamp, "durable_writes"));
        cf.addColumn(Column.create(strategyClass.getName(), timestamp, "strategy_class"));
        cf.addColumn(Column.create(json(strategyOptions), timestamp, "strategy_options"));
    
        for (CFMetaData cfm : cfMetaData.values())
            cfm.toSchema(rm, timestamp);
    
        return rm;
    }
    </code>



And `SystemTable.SCHEMA_COLUMNFAMILIES_CF`. Quite a few ColumnFamily level options (only required options are listed). For property that can be null (and can be changed), we insert tombstones, to make sure we don't keep a property the user has removed.




    
    <code>/* org.apache.cassandra.config.CFMetaData */
    public void toSchema(RowMutation rm, long timestamp)
    {
        toSchemaNoColumns(rm, timestamp);
    
        for (ColumnDefinition cd : column_metadata.values())
            cd.toSchema(rm, cfName, getColumnDefinitionComparator(cd), timestamp);
    }
    
    /* org.apache.cassandra.config.CFMetaData */
    private void toSchemaNoColumns(RowMutation rm, long timestamp)
    {
        ColumnFamily cf = rm.addOrGet(SystemTable.SCHEMA_COLUMNFAMILIES_CF);
        int ldt = (int) (System.currentTimeMillis() / 1000);
    
        Integer oldId = Schema.instance.convertNewCfId(cfId);
    
        if (oldId != null) // keep old ids (see CASSANDRA-3794 for details)
            cf.addColumn(Column.create(oldId, timestamp, cfName, "id"));
    
        cf.addColumn(Column.create(cfType.toString(), timestamp, cfName, "type"));
        cf.addColumn(Column.create(comparator.toString(), timestamp, cfName, "comparator"));
        if (subcolumnComparator != null)
            cf.addColumn(Column.create(subcolumnComparator.toString(), timestamp, cfName, "subcomparator"));
        ... // to name but a few
    }
    </code>



Finally, `SystemTable.SCHEMA_COLUMNS_CF`. **ColumnDefition** doesn't show up as frequently as KSMetaData and CFMetaData but it's just "ColumnMetaData".




    
    <code>/* org.apache.cassandra.config.ColumnDefition */
    public void toSchema(RowMutation rm, String cfName, AbstractType<?> comparator, long timestamp)
    {
        ColumnFamily cf = rm.addOrGet(SystemTable.SCHEMA_COLUMNS_CF);
        int ldt = (int) (System.currentTimeMillis() / 1000);
    
        cf.addColumn(Column.create(validator.toString(), timestamp, cfName, comparator.getString(name), "validator"));
        ... // index options.
    }
    </code>



Note that all the above are all `toSchema` methods, and like `toThrift` we have `fromSchema` methods which deserialize column family records into structures, namely, KSMetaData, CFMetaData and ColumnDefinition.



In our example, we didn't add any column families or columns, so only the keyspace level updates are actually performed. We will revisit the column family level updates in the next section.



Now all the updates have been put into the following RowMutation `modications` map:




    
    <code>/* org.apache.cassandra.db.RowMutation */
    protected Map<UUID, ColumnFamily> modifications = new HashMap<UUID, ColumnFamily>();
    </code>



We are now ready to make changes to all live hosts (including localhost).




    
    <code>/* org.apache.cassandra.service.MigrationManager */
    
    private static void announce(RowMutation schema)
    {
        FBUtilities.waitOnFuture(announce(Collections.singletonList(schema)));
    }
    
    private static Future<?> announce(final Collection<RowMutation> schema)
    {
        Future<?> f = StageManager.getStage(Stage.MIGRATION).submit(new Callable<Object>()
        {
            public Object call() throws Exception
            {
                DefsTable.mergeSchema(schema);
                return null;
            }
        });
    
        for (InetAddress endpoint : Gossiper.instance.getLiveMembers())
        {
            if (endpoint.equals(FBUtilities.getBroadcastAddress()))
                continue; // we've delt with localhost already
    
            // don't send migrations to the nodes with the versions older than < 1.1
            if (MessagingService.instance().getVersion(endpoint) < MessagingService.VERSION_11)
                continue;
    
            pushSchemaMutation(endpoint, schema);
        }
        return f;
    }
    </code>



We send out messages with new schema as the payload. The remote host will recognize the `DEFINITIONS_UPDATE` verb and the corresponding registered VerbHandler will handle it.




    
    <code>/* org.apache.cassandra.service.MigrationManager */
    private static void pushSchemaMutation(InetAddress endpoint, Collection<RowMutation> schema)
    {
        MessageOut<Collection<RowMutation>> msg = new MessageOut<Collection<RowMutation>>(MessagingService.Verb.DEFINITIONS_UPDATE,
                                                                                          schema,
                                                                                          MigrationsSerializer.instance);
        MessagingService.instance().sendOneWay(msg, endpoint);
    }
    </code>



Look it up in `org.apache.cassandra.service.StorageService`:




    
    <code> /* org.apache.cassandra.db.DefinitionsUpdateVerbHandler */
     MessagingService.instance().registerVerbHandlers(MessagingService.Verb.DEFINITIONS_UPDATE, new DefinitionsUpdateVerbHandler());
    </code>



Its `doVerb` exposes how it will handle the message. Not surprisingly, exactly what we do locally, `DefsTable.mergeSchema` which we will visit next.




    
    <code>public void doVerb(final MessageIn<Collection<RowMutation>> message, String id)
    {
        logger.debug("Received schema mutation push from " + message.from);
    
        StageManager.getStage(Stage.MIGRATION).submit(new WrappedRunnable()
        {
            public void runMayThrow() throws Exception
            {
                if (message.version < MessagingService.VERSION_11)
                {
                    logger.error("Can't accept schema migrations from Cassandra versions previous to 1.1, please upgrade first");
                    return;
                }
                DefsTable.mergeSchema(message.payload);
            }
        });
    }
    </code>



`mutation.apply` will write to CommitLog and Memtable. Then, we force flush the three schema column families to SSTables. Finally, update the global schema (`org.apache.cassandra.config.Schema`).




    
    <code>/* org.apache.cassandra.service.MigrationManager */
    public static synchronized void mergeSchema(Collection<RowMutation> mutations) throws ConfigurationException, IOException
    {
        // current state of the schema
        Map<DecoratedKey, ColumnFamily> oldKeyspaces = SystemTable.getSchema(SystemTable.SCHEMA_KEYSPACES_CF);
        Map<DecoratedKey, ColumnFamily> oldColumnFamilies = SystemTable.getSchema(SystemTable.SCHEMA_COLUMNFAMILIES_CF);
    
        for (RowMutation mutation : mutations)
            mutation.apply();
    
        if (!StorageService.instance.isClientMode())
            flushSchemaCFs();
    
        Schema.instance.updateVersionAndAnnounce();
    
        // with new data applied
        Map<DecoratedKey, ColumnFamily> newKeyspaces = SystemTable.getSchema(SystemTable.SCHEMA_KEYSPACES_CF);
        Map<DecoratedKey, ColumnFamily> newColumnFamilies = SystemTable.getSchema(SystemTable.SCHEMA_COLUMNFAMILIES_CF);
    
        Set<String> keyspacesToDrop = mergeKeyspaces(oldKeyspaces, newKeyspaces);
        mergeColumnFamilies(oldColumnFamilies, newColumnFamilies);
    
        // it is safe to drop a keyspace only when all nested ColumnFamilies where deleted
        for (String keyspaceToDrop : keyspacesToDrop)
            dropKeyspace(keyspaceToDrop);
    }
    </code>



We calculate the difference between old and new states (note that entriesOnlyLeft() will be always empty since old column families are dropped).




    
    <code>/* org.apache.cassandra.service.MigrationManager */
    private static Set<String> mergeKeyspaces(Map<DecoratedKey, ColumnFamily> old, Map<DecoratedKey, ColumnFamily> updated)
    {
        MapDifference<DecoratedKey, ColumnFamily> diff = Maps.difference(old, updated);
    
        for (Map.Entry<DecoratedKey, ColumnFamily> entry : diff.entriesOnlyOnRight().entrySet())
        {
            ColumnFamily ksAttrs = entry.getValue();
    
            if (!ksAttrs.isEmpty())
                addKeyspace(KSMetaData.fromSchema(new Row(entry.getKey(), entry.getValue()), Collections.<CFMetaData>emptyList()));
        }
        ...
    }
    </code>



We don't create any disk files when adding keyspace which is different from adding column families. Instead, we only init a `Table` object and store it into global schema.




    
    <code>private static void addKeyspace(KSMetaData ksm)
    {
        assert Schema.instance.getKSMetaData(ksm.name) == null;
        Schema.instance.load(ksm);
    
        if (!StorageService.instance.isClientMode())
            Table.open(ksm.name);
    }
    </code>



Remark the process is going at all the hosts so they each end up with a new `Table` object.



Since we have yet created any column families, the `mergeColumnFamilies` method does nothing in effect. Hence, we'll talk about it thoroughly in the next section.





# What happens when we create a new column family?



And how about:




    
    <code>[default@unknown] use Twissandra;
    [default@Twissandra] create column family User with comparator = UTF8Type;
    </code>



Our Thrift processor (CassandraServer) makes the corresponding procedure call.




    
    <code>/* org.apache.cassandra.service.CassandraServer */
    public String system_add_column_family(CfDef cf_def)
    throws InvalidRequestException, SchemaDisagreementException, TException
    {
        logger.debug("add_column_family");
        state().hasColumnFamilySchemaAccess(Permission.WRITE);
    
        try
        {
            cf_def.unsetId(); // explicitly ignore any id set by client (Hector likes to set zero)
            CFMetaData cfm = CFMetaData.fromThrift(cf_def);
            if (cfm.getBloomFilterFpChance() == null)
                cfm.bloomFilterFpChance(CFMetaData.DEFAULT_BF_FP_CHANCE);
            cfm.addDefaultIndexNames();
            MigrationManager.announceNewColumnFamily(cfm);
            return Schema.instance.getVersion().toString();
        }
        catch (ConfigurationException e)
        {
            InvalidRequestException ex = new InvalidRequestException(e.getMessage());
            ex.initCause(e);
            throw ex;
        }
    }
    </code>



Creating a column family is the "subset" of creating a keyspace, so we'll start from where we leave off in the last section.




    
    <code>/* org.apache.cassandra.db.DefsTable */
    private static void mergeColumnFamilies(Map<DecoratedKey, ColumnFamily> old, Map<DecoratedKey, ColumnFamily> updated)
            throws ConfigurationException, IOException
    {
        MapDifference<DecoratedKey, ColumnFamily> diff = Maps.difference(old, updated);
    
        // check if any new Keyspaces with ColumnFamilies were added.
        for (Map.Entry<DecoratedKey, ColumnFamily> entry : diff.entriesOnlyOnRight().entrySet())
        {
            ColumnFamily cfAttrs = entry.getValue();
    
            if (!cfAttrs.isEmpty())
            {
               Map<String, CFMetaData> cfDefs = KSMetaData.deserializeColumnFamilies(new Row(entry.getKey(), cfAttrs));
    
               for (CFMetaData cfDef : cfDefs.values())
                    addColumnFamily(cfDef);
            }
        }
        ...
    }
    </code>



We update KSMetaData by concatenating existing CFMetaData with that of new column family. Then we init a new column family.




    
    <code>/* org.apache.cassandra.db.DefsTable */
    private static void addColumnFamily(CFMetaData cfm)
    {
        assert Schema.instance.getCFMetaData(cfm.ksName, cfm.cfName) == null;
        KSMetaData ksm = Schema.instance.getTableDefinition(cfm.ksName);
        ksm = KSMetaData.cloneWith(ksm, Iterables.concat(ksm.cfMetaData().values(), Collections.singleton(cfm)));
    
        Schema.instance.load(cfm);
    
        // make sure it's init-ed w/ the old definitions first,
        // since we're going to call initCf on the new one manually
        Table.open(cfm.ksName);
    
        Schema.instance.setTableDefinition(ksm);
    
        if (!StorageService.instance.isClientMode())
            Table.open(ksm.name).initCf(cfm.cfId, cfm.cfName, true);
    }
    </code>



We adds a column family to internal structures, ends up creating disk files. As it is with new keyspace, we have new disk files at every host.




    
    <code>/* org.apache.cassandra.db.Table */
    public void initCf(UUID cfId, String cfName, boolean loadSSTables)
    {
        if (columnFamilyStores.containsKey(cfId))
        {
            // this is the case when you reset local schema
            // just reload metadata
            ColumnFamilyStore cfs = columnFamilyStores.get(cfId);
            assert cfs.getColumnFamilyName().equals(cfName);
    
            cfs.metadata.reload();
            cfs.reload();
        }
        else
        {
            columnFamilyStores.put(cfId, ColumnFamilyStore.createColumnFamilyStore(this, cfName, loadSSTables));
        }
    }
    </code>



A ColumnFamilyStore has a **DataTracker** which keeps a View holding the current MemTable.




    
    <code>/* org.apache.cassandra.db.ColumnFamilyStore */
    private ColumnFamilyStore(Table table,
                          String columnFamilyName,
                          IPartitioner partitioner,
                          int generation,
                          CFMetaData metadata,
                          Directories directories,
                          boolean loadSSTables)
    {
         ...
         data = new DataTracker(this);
         ...
    }
    
    /* org.apache.cassandra.db.DataTracker */
    public DataTracker(ColumnFamilyStore cfstore)
    {
        this.cfstore = cfstore;
        this.view = new AtomicReference<View>();
        this.init();
    }
    
    /* org.apache.cassandra.db.DataTracker */
    void init()
    {
        view.set(new View(new Memtable(cfstore),
                          Collections.<Memtable>emptySet(),
                          Collections.<SSTableReader>emptyList(),
                          Collections.<SSTableReader>emptySet(),
                          SSTableIntervalTree.empty()));
    }
    </code>



The directory layout is the following:




    
    <code> <path_to_data_dir>/ks/cf1/ks-cf1-hb-1-Data.db
                          /cf2/ks-cf2-hb-1-Data.db
    </code>



In our example:




    
    <code> <path_to_data_dir>/Twissandra/User/ 
    </code>



All the details of handling of data directory are encapsulated into `org.apache.cassandra.db.Directories`.



