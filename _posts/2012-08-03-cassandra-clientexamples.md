---
author: manuzhang
comments: true
date: 2012-08-03 06:38:06+00:00
layout: post
slug: cassandra-clientexamples
title: Cassandra ClientExamples
wordpress_id: 276
categories:
- database
- 实习笔记
tags:
- Cassandra
- Java
- Thrift
---

Just got my feet wet in writing a Cassandra client via [Thrift API](http://wiki.apache.org/cassandra/API) so I'd like to share my experience here.



Thrift API is to Cassandra what [JDBC](http://en.wikipedia.org/wiki/JDBC) ([ODBC](http://en.wikipedia.org/wiki/Odbc)) is to a RDBMS. It provides methods for querying and updating data in Cassandra.



I intended to explore into Cassandra code base, and thus I [ran it in Eclipse](http://wiki.apache.org/cassandra/RunningCassandraInEclipse). My Cassandra version is **1.2.0-SNAPSHOT**, so I have trouble in compiling or running examples provided in the [Definitive Guide](http://www.ppurl.com/2010/12/cassandra-the-definitive-guide.html) (for **0.7**) and on the [official website](http://wiki.apache.org/cassandra/ClientExamples) (for **1.0**). The basic idea is the same though. Let's get started.



<!-- more -->



# Prerequisites



First of all, I created a new Java project in Eclipse and added to my classpath the needed JARs:







  * libthrift-0.7.0.jar


  * log4j-1.2.16.jar


  * slf4j-api-1.6.1.jar


  * slf4j-log4j12-1.6.1.jar


  * apache-cassandra-1.2.0-SNAPSHOT.jar



Where to get the JARs? From the Cassandra I cloned.



I found first four in [cci]cassandra-trunk/lib[/cci] and generated the last one in terminal:

[cc]
ant jar
[/cc]

This performed a complete build and output the JAR file into the [cci]cassandra-trunk/build[/cci] directory.





# And the Code



[cc lang="java"]
package org.apache.cassandra.examples;

import java.io.UnsupportedEncodingException;
import java.nio.ByteBuffer;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

import org.apache.cassandra.thrift.Cassandra;
import org.apache.cassandra.thrift.CfDef;
import org.apache.cassandra.thrift.Column;
import org.apache.cassandra.thrift.ColumnOrSuperColumn;
import org.apache.cassandra.thrift.ColumnParent;
import org.apache.cassandra.thrift.ColumnPath;
import org.apache.cassandra.thrift.ConsistencyLevel;
import org.apache.cassandra.thrift.InvalidRequestException;
import org.apache.cassandra.thrift.KeyRange;
import org.apache.cassandra.thrift.KeySlice;
import org.apache.cassandra.thrift.KsDef;
import org.apache.cassandra.thrift.NotFoundException;
import org.apache.cassandra.thrift.SchemaDisagreementException;
import org.apache.cassandra.thrift.SlicePredicate;
import org.apache.cassandra.thrift.SliceRange;
import org.apache.cassandra.thrift.TimedOutException;
import org.apache.cassandra.thrift.UnavailableException;
import org.apache.thrift.TException;
import org.apache.thrift.protocol.TBinaryProtocol;
import org.apache.thrift.protocol.TProtocol;
import org.apache.thrift.transport.TFramedTransport;
import org.apache.thrift.transport.TSocket;
import org.apache.thrift.transport.TTransport;
import org.junit.Test;

public class SimpleWriteRead 
{

    private static final String HOST = "localhost";
    private static final int PORT = 9160;
    private static final String UTF8 = "utf8";
    private static final ConsistencyLevel CL = ConsistencyLevel.ONE;

    @Test
    public void testWriteRead() 
            throws UnsupportedEncodingException, InvalidRequestException, TException, UnavailableException, TimedOutException, SchemaDisagreementException, NotFoundException 
    {
        // connect to server
        TTransport tr = new TFramedTransport(new TSocket(HOST, PORT));
        tr.open();

        TProtocol proto = new TBinaryProtocol(tr);
        Cassandra.Client client = new Cassandra.Client(proto);

        // create keyspace 
        String keyspace = "Keyspace1";
        KsDef ksDef = new KsDef();
        ksDef.setName(keyspace);
        ksDef.setStrategy_class("org.apache.cassandra.locator.SimpleStrategy");
        Map<String, String> options = new HashMap<String, String>();
        int repFactor = 1;
        options.put("replication_factor", Integer.toString(repFactor));
        ksDef.setStrategy_options(options);

        // add column family
        String columnFamily = "Standard1";
        CfDef cfDef = new CfDef(keyspace, columnFamily);

        List<CfDef> cfDefs = new ArrayList<CfDef>();
        cfDefs.add(cfDef);
        ksDef.setCf_defs(cfDefs);

        try
        {
            client.system_drop_keyspace(keyspace);
        }
        catch (InvalidRequestException e)
        {
        }
        finally
        {
            client.system_add_keyspace(ksDef);
        }

        // insert data
        client.set_keyspace(keyspace);
        String keyUserId = "1024"; // row id

        long timestamp = System.currentTimeMillis();
        String name = "manu";
        String age = "21";

        Column nameColumn = new Column(ByteBuffer.wrap("name".getBytes(UTF8)));
        nameColumn.setValue(name.getBytes(UTF8));
        nameColumn.setTimestamp(timestamp);

        Column ageColumn = new Column(ByteBuffer.wrap("age".getBytes(UTF8)));
        ageColumn.setValue(age.getBytes(UTF8));
        ageColumn.setTimestamp(timestamp);

        ColumnParent columnParent = new ColumnParent(columnFamily);
        ByteBuffer key = ByteBuffer.wrap(keyUserId.getBytes(UTF8));
        client.insert(key, columnParent, nameColumn, CL);
        client.insert(key, columnParent, ageColumn, CL);

        // read columns
        ColumnPath columnPath = new ColumnPath(columnFamily);

        columnPath.setColumn(ByteBuffer.wrap("name".getBytes(UTF8)));
        nameColumn = client.get(key, columnPath, CL).getColumn();
        System.out.println(new String(nameColumn.getName(), UTF8) + ":" + new String(nameColumn.getValue(), UTF8) + ":" + nameColumn.getTimestamp());

        columnPath.setColumn(ByteBuffer.wrap("age".getBytes(UTF8)));
        ageColumn = client.get(key, columnPath, CL).getColumn();
        System.out.println(new String(ageColumn.getName(), UTF8) + ":" + new String(ageColumn.getValue(), UTF8) + ":" + ageColumn.getTimestamp());

        // create a slice predicate representing the columns to read
        // start and finish are the range of columns -- here, all
        SlicePredicate predicate = new SlicePredicate();
        predicate.setSlice_range(new SliceRange(ByteBuffer.wrap(new byte[0]), ByteBuffer.wrap(new byte[0]), false, 100));
        List<ColumnOrSuperColumn> columnsByKey = client.get_slice(key, columnParent, predicate, CL);
        for (ColumnOrSuperColumn columnByKey : columnsByKey)
        {
            Column column = columnByKey.getColumn();
            System.out.println(new String(column.getName(), UTF8) + ":" + new String(column.getValue()) + ":" + column.getTimestamp());
        }

        // get all keys
        KeyRange keyRange = new KeyRange(100);
        keyRange.setStart_key(new byte[0]);
        keyRange.setEnd_key(new byte[0]);
        List<KeySlice> keySlices = client.get_range_slices(columnParent, predicate, keyRange, CL);
        System.out.println(keySlices.size());
        for (KeySlice ks : keySlices)
        {
            System.out.println(new String(ks.getKey()));
        }

        tr.close();
    }
}
[/cc]

Don't worry about the long list of exceptions. Eclipse will handle that. I did copy most codes (comments) from the two aforementioned sources. The only original work is to set strategy class and its replication factor option for the keyspace. I guess this has been mandatory since **1.1** ([CASSANDRA-4294](https://issues.apache.org/jira/browse/CASSANDRA-4294)).





# RPC



Remote Procedural Call (RPC) in Cassandra is implemented against [Apache Thrift](http://en.wikipedia.org/wiki/Apache_Thrift) framework:



![apache_thrift_arch](http://upload.wikimedia.org/wikipedia/en/thumb/d/df/Apache_Thrift_architecture.png/331px-Apache_Thrift_architecture.png)





# Outputs



[cc]
 name:manu:1343974116447
 age:21:1343974116447
 age:21:1343974116447
 name:manu:1343974116447
 1
 1024
[/cc]

From the Datastax documentation of [using CLI](http://www.datastax.com/docs/1.0/dml/using_cli):





<blockquote>
  Cassandra stores all data internally as hex byte arrays by default. If you do not specify a default row key validation class, column comparator and column validation class when you define the column family, Cassandra CLI will expect input data for row keys, column names, and column values to be in hex format (and data will be returned in hex format).


  
  To pass and return data in human-readable format, you can pass a value through an encoding function.


</blockquote>



I suppose it is the same with Cassandra Thrift API and my encoding here is UTF8.



