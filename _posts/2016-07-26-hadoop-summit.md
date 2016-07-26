---
comments: true
layout: post
title:  Streaming at Hadoop Summit 2016
---

This blog has been biased towards *Streaming* although I meant to write all things *Big Data*. This time I will look back at Streaming sessions of Hadoop Summit 2016 is held in San Jose from June 28 to June 30.  I didn't go to the event so all my take is based on  videos/slides of the *IoT and Streaming* track. You may checkout Zhang Zhe's [Notes from Hadoop Summit 2016](http://zhe-thoughts.github.io/2016/07/11/Hadoop-Summit/) for HDFS news and [Hadoop Summit 2016: The Growth Accelerates - Hortonworks](http://hortonworks.com/blog/hadoop-summit-2016-growth-accelerates/) from business's perspectives.

### IoT Streaming

* [Connected Vehicle Data Platform](http://www.slideshare.net/HadoopSummit/connected-vehicle-data-platform) introduced Ford Motor's data platform with interesting data. 

	> A single Vehicle can generate 25GB of Controller Area Network (CAN) in a hour. 

* [Building a Smarter Home with Apache NiFi and Spark](http://www.slideshare.net/HadoopSummit/building-a-smarter-home-with-apache-nifi-and-spark) builds around a mixed environment of edge, data center and cloud where **Apache Nifi** plays as a connector all over places. 

### Streaming Engines

* While **Flink** author Kostas Tzoumas talked more about use cases at [Streaming in the Wild with Apache Flink](http://www.slideshare.net/KostasTzoumas/streaming-in-the-wild-with-apache-flink-63790942), Stephan Ewen dived deeper into technique details especially state management at [The Stream Processor as a Database Apache Flink](http://www.slideshare.net/HadoopSummit/the-stream-processor-as-a-database-apache-flink).

* [Next Gen Big Data Analytics with Apache Apex](http://www.slideshare.net/HadoopSummit/next-gen-big-data-analytics-with-apache-apex) gave a general introduction to stream processing engine, **Apache Apex**, from DataTorrent. 

* **Storm** PMC Chair Taylor Goetz talked about [The Future of Apache Storm](http://www.slideshare.net/HadoopSummit/the-future-of-apache-storm-63920895). Although about *future*, I found it the best documentation to learn about current status of Storm (1.0.1) so far. A major feature in the future is [Resource Aware Scheduling](http://www.slideshare.net/HadoopSummit/resource-aware-scheduling-in-apache-spark)

* Guozhang Wang talked about how **Kafka Streams** dealt with *streaming processing hard parts* in [Stream Processing made simple with Kafka]( 
http://www.slideshare.net/HadoopSummit/stream-processing-made-simple-with-kafka).

* **Apache Calcite** author Julian Hyde introduced [Streaming SQL](http://www.slideshare.net/HadoopSummit/streaming-sql-63920557), which is being integrated into Storm, Flink and Samza as SQL over streaming solution.

* With various batch and streaming engines, there is one to unify all, [**Apache Beam**: A unified model for batch and stream processing data](http://www.slideshare.net/HadoopSummit/apache-beam-a-unified-model-for-batch-and-stream-processing-data)

### Streaming platforms

* Streaming at Symantec has been used to process log and metrics data. They shared about their pipeline (Storm + ELK) and how they handled the influx issue in [In Flux Limiting for a multi-tenant logging service](http://www.slideshare.net/HadoopSummit/in-flux-limiting-for-a-multitenant-logging-service).

* Another sharing from Symantec which took close look at their [End to End Processing of 3.7 Million Telemetry Events per Second using Lambda Architecture 
](http://www.slideshare.net/HadoopSummit/end-to-end-processing-of-37-million-telemetry-events-per-second-using-lambda-architecture). The deck is full of practical contents like tuning parameters and benchmark results on Kafka, Storm(Trident), etc.

* [Lambda-less Stream Processing @Scale in LinkedIn]( 
http://www.slideshare.net/HadoopSummit/lambdaless-stream-processing-scale-in-linkedin) went through how hard problems (*accurracy* and *reprocessing*) in stream processing had been solved without lambda architecture in Linkedin. The solution is based on **Apache Samza** and influenced by Google's [Millwheel](http://research.google.com/pubs/pub41378.html).

### Summary

Almost all Apache Streaming Engines made their presence at Hadoop Summit although it is biased towards Apache Nifi for obvious reasons. One can get a great bird view of existing streaming problems and solutions going through the contents. 