---
comments: true
layout: post
title: Instrument NameNode metrics with Hadoop Metrics2
--- 

Recently at work, we need to collect the metrics of users accessing HDFS, e.g. how many times a user has read a file, to help users adjust their storage policies (For more on background, please refer to [HDFS-7343](https://issues.apache.org/jira/browse/HDFS-7343) and [Smart Storage Management](https://github.com/Intel-bigdata/SSM)). Unfortunately, that is not available in the [existing (Hadoop 2.7.3) metrics](https://hadoop.apache.org/docs/r2.7.3/hadoop-project-dist/hadoop-common/Metrics.html) which means we have to hack NameNode (who knows it when users access HDFS files) ourselves. Fortunately, we don't have to start from scratch since Hadoop already provides a pluggable metrics framework, [Hadoop Metrics2](https://hadoop.apache.org/docs/r2.7.3/api/org/apache/hadoop/metrics2/package-summary.html),

> The framework provides a variety of ways to implement metrics instrumentation easily via the simple MetricsSource interface or the even simpler and more concise and declarative metrics annotations. The consumers of metrics just need to implement the simple MetricsSink interface. Producers register the metrics sources with a metrics system, while consumers register the sinks. A default metrics system is provided to marshal metrics from sources to sinks based on (per source/sink) configuration options...

Specifically for our requirement, we need to

1. Implement a `MetricsSource` inside NameNode that record users accessing file. 
2. Implement a `MetricsSink` that write the metrics somewhere (e.g. HDFS).
3. Finally, hook them together.

The default metrics system is a singleton so that we have to add our `MetricsSource` into the existing `NameNodeMetrics`. 

Here is the big picture of the metrics flow. The blue parts are already provided while the red parts should be implemented.

![hadoop_metrics2](https://goo.gl/v9MuvS){:height="200px" width="650px"}



## MetricsSource

As said in the doc, there are two ways to write a `MetricsSource`. The simpler and more limited one is `@Metrics` annotation.

### `@Metrics` annotation
   
   For example, the [NameNodeMetrics](https://github.com/apache/hadoop/blob/branch-2.7.3/hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/server/namenode/metrics/NameNodeMetrics.java), where`@Metric` is used to indict a metrics source.
   
   ```java
   @Metrics(name="NameNodeActivity", about="NameNode metrics", context="dfs")
   public class NameNodeMetrics {
     ...
     @Metric MutableCounterLong createFileOps;
     @Metric MutableCounterLong filesCreated;
     @Metric MutableCounterLong filesAppended;
     
     FileAccessMetrics fileAccessMetrics;
     ...
   }
   ```

   The limitation is the class must have at least one `@Metric` field, whose class has to extend `MutableMetric`.  Other than that, we are free to add any `MetricsSource`, for instance, the `FileAccessMetrics` we are going to implement.

### Implementing `MetricsSource`

   ```java
   public class FileAccessMetrics implements MetricsSource {
     public static final String NAME = "FileAccessMetrics";
     public static final String DESC = "FileAccessMetrics";
     public static final String CONTEXT_VALUE ="file_access";

     private List<Info> infos = new LinkedList<>();

     public static FileAccessMetrics create(MetricsSystem ms) {
       return ms.register(NAME, DESC, new FileAccessMetrics());
    }

     public synchronized void addMetrics(String path, String user, long time) {
       infos.add(new Info(path, user, time));
     }

     @Override
     public void getMetrics(MetricsCollector collector, boolean all) {
       for (Info info: infos) {
         MetricsRecordBuilder rb = collector.addRecord(info).setContext(CONTEXT_VALUE);
         rb.addGauge(info, 1);
       }
       infos.clear();
     }
   ```
   
   Distilling it,
   
   * `NameNodeMetrics` will `create` this `FileAccessMetrics`.
   * Whenever `DFSClient` opens a read call, `NameNode` will `addMetrics(Info(path, user, time)` to the list. 
   * The `MetricsSystem` will periodically `getMetrics` from the list and put onto its internal queue.
   * The `MetricsRecordBuilder` expect a numerical value so we do a small trick by storing the `Info` into the record name and setting the value to `1`. 
   * The `CONTEXT_VALUE` will be used later to identify the record for write.


## MetricsSink

Periodically, `MetricsSystem` will poll its internal queue 
and `putMetrics` to `MetricsSink`, which can write it out as  follows. 

```java
public class FileAccessMetrics implements MetricsSource {
  ...
  public static class Writer implements MetricsSink, Closeable {
    ...
    @Override
    public void putMetrics(MetricsRecord record) {
      ...
      for (AbstractMetric metric : record.metrics()) {
        currentOutStream.print(metric.name() + ":" + metric.value());
      }
      ...
    }
    ...
  } 
  ...
}  
```

Note that Hadoop 3.x already packs a bunch of useful sinks

  * RollingFileSystemSink (Hadoop FileSystem)
  * KafkaSink
  * GraphiteSink
  * StatsDSink

## Now, hook them together

Put the following configurations into either `hadoop-metrics2-namenode.properties` or `hadoop-metrics2.properties`. 

```
namenode.sink.file_access.class=org.apache.hadoop.hdfs.server.namenode.metrics.FileAccessMetrics$Writer
namenode.sink.file_access.context=file_access
```

* `namenode` is the prefix with which `NameNodeMetrics` initialize the metrics system.
* The metrics system will firstly try to load `hadoop-metrics2-[prefix].properties` and fall back to `hadoop-metrics2.properties` if not found.
* The context value is exactly `FileAccessMetrics$CONTEXT_VALUE` such that `MetricsSystem` are able to filter out other `NameNodeMetrics` sources and only send `FileAccessMetrics` to our sink.  

## Summary

This post describes the architecture and usage of Hadoop metrics2 through an example to instrument user accessing HDFS files. However, I cannot cover all the features since I haven't tried them out myself so please refer to the [official documentation](http://hadoop.apache.org/docs/r2.7.3/api/org/apache/hadoop/metrics2/package-summary.html).

