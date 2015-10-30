# Storm Basics - Topology

The processing logic in MapReduce is defined as two simple functions, `map` and `reduce`. To express complex queries, uses need to chain multiple MapReduce functions together. Dryad generalizes it to dataflow, a Directed Acyclic Graph (DAG). Processing logics in Storm is also expressed as a DAG, called Topology. The source vertex is a Spout, and others Bolt. The edge is stream. 

They are all defined as thrift objects.

Users define a topology, which will be submit to Nimbus through thrift API.

The basic interface for Spout is `ISpout` and basic interface for Bolt is `IBolt`

## User Topology



## System Topology

The user topology is not the final topology executed on Storm cluster. Storm will add system bolts if configured

### AckerBolt

### MetricsConsumerBolt

Storm allows users to register a custom metrics consumer to receive topology metrics. Also, Storm has given an example as `LoggingMetricsConsumer`. System sends metrics data to `MetricsConsumerBolt` where the metrics consumer is instantiated.

```yaml
## Metrics Consumers
topology.metrics.consumer.register:
  - class: "backtype.storm.metric.LoggingMetricsConsumer"
  parallelism.hint: 1
```


### SystemBolt

A special `SystemBolt` is conceived to export worker stats.

```yaml
topology.builtin.metrics.bucket.size.secs: 60
```
