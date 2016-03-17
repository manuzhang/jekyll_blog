---
comments: true
layout: post
title: Streaming at Spark Summit East 2016
---

Spark Summit East 2016 is held in NYC last month. 
Databricks already had a [look back](https://databricks.com/blog/2016/02/18/a-look-back-at-spark-summit-east-2016-thank-you-nyc.html), and I'm going to focus on the (Spark) streaming part here. 

## Highlights

The most interesting ones are 

1. [Spark Streaming and Iot](https://spark-summit.org/east-2016/events/spark-streaming-and-iot/) 
2. [Online Security Analytics on Large Scale Video Survellance System](https://spark-summit.org/east-2016/events/online-security-analytics-on-large-scale-video-surveillance-system/) 
3. [Clickstream Analysis with Spark—Understanding Visitors in Realtime
](https://spark-summit.org/east-2016/events/clickstream-analysis-with-spark-understanding-visitors-in-realtime/)

### Spark Streaming and Iot

Mike Freedman, CEO and Co-Founder of [iobeam](http://www.iobeam.com/), mainly talked about the challenges in applying Spark to IoT.

![challenges_applying_spark_iot](http://image.slidesharecdn.com/1gmikefreedman-160224030556/95/spark-streaming-and-iot-by-mike-freedman-4-638.jpg?cb=1456283190)

I like this talk because these challenges are quite general. It's unclear how iobeam solved them with Spark Streaming which only supports data arrival time. iobeam is a data analysis platform designed for IoT. I really enjoy their websites which put codes side-by-side with use cases. 
  

### Online Security Analytics on Large Scale Video Survellance System

This is from EMC Video Analytics Data Lake, where Spark Streaming is used for online video processing and detection. 

![online_video_processing](http://image.slidesharecdn.com/3mhyucao-160224032524/95/online-security-analytics-on-large-scale-video-surveillance-system-by-yu-cao-and-xiaoyan-guo-11-638.jpg?cb=1456284399)

Streaming application serves to feed offline model training which is in turn used to realtime detection.

### Clickstream Analysis with Spark—Understanding Visitors in Realtime

The talk is really about the architecture evolution from "Larry & Friends" (Oracle) to "Hadoop & Friends" (HDFS, Hive), from Kappa-Architecture to Lambda-Architecture, and finally Mu-Architecture all based on Spark.

![connection_streaming_batch](http://image.slidesharecdn.com/9scjadersberger-160224034337/95/clickstream-analysis-with-sparkunderstanding-visitors-in-realtime-by-josef-adersberger-19-638.jpg?cb=1456285460)

Note that realtime here means 15 mins so a low latency streaming engine like Storm is overengineered. It's, however, a sweet spot for Spark Streaming given the other components in the system are also based on Spark. 


## Core

Spark 2.0 will add an infinite Dataframes API for Spark Streaming, unified with the existing Dataframes API for batch processing.  Event-time aggregations will finally arrive in Spark Streaming.

1. [The Future of Real-Time in Spark](https://spark-summit.org/east-2016/events/keynote-day-3/) by Reynold Xin, Databricks
2. [Structuring Spark Dataframes, Datasets and Streaming](https://spark-summit.org/east-2016/events/structuring-spark-dataframes-datasets-and-streaming/) by Michael Amburst, Databricks.

Meanwhile, Back pressure and Elastic Scaling are two important features under development.

1. [Reactive Streams, linking Reactive Application to Spark Streaming](https://spark-summit.org/east-2016/events/building-robust-scalable-and-adaptive-applications-on-spark-streaming/) by Luc Bourlier, Lightbend.
2. [Building Robust, Scalable and Adaptive Applications on Spark Streaming](https://spark-summit.org/east-2016/events/reactive-streams-linking-reactive-application-to-spark-streaming/) by Tathagata Das, Databricks.

## Connectors

1. [Realtime Risk Management Using Kafka, Python, and Spark Streaming](https://spark-summit.org/east-2016/events/realtime-risk-management-using-kafka-python-and-spark-streaming/) at Shopify. 
2. [Building Realtime Data Pipelines with Kafka Connect and Spark Streaming](https://spark-summit.org/east-2016/events/building-realtime-data-pipelines-with-kafka-connect-and-spark-streaming/) by Confluent. This is more about [Kafka connnect](http://docs.confluent.io/2.0.0/connect/) than Spark Streaming. 

## Use cases 

Other less interesting use cases

1. [Interactive Visualization of Streaming Data Powered by Spark](https://spark-summit.org/east-2016/events/interactive-visualization-of-streaming-data-powered-by-spark/) introduces streaming data visualization at [Zoomdata](http://www.zoomdata.com/).
2. [Online Predictive Modeling of Fraud Schemes from Mulitple Live Streams](https://spark-summit.org/east-2016/events/online-predictive-modeling-of-fraud-schemes-from-mulitple-live-streams/) at Atigeo.
3. [Using Spark to Analyze Activity and Performance in High Speed Trading Environments](https://spark-summit.org/east-2016/events/using-spark-to-analyze-activity-and-performance-in-high-speed-trading-environments/)


## References

1. https://spark-summit.org/east-2016/schedule/
