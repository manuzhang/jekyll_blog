---
comments: true
layout: post
title: Flink Forward SF 2017 Readings
---

[Flink Forward](http://sf.flink-forward.org/), "The premier conference on Apache Flink®", just took place in San Francisco. All the [slides](https://www.slideshare.net/FlinkForward) and [videos](https://www.youtube.com/playlist?list=PLDX4T_cnKjD2UC6wJr_wRbIvtlMtkc-n2) are available now. The conference was both abundant in practical experiences and technical details. After going through all the slides, I'd like to share some interesting contents that you can't find on [http://flink.apache.org/](http://flink.apache.org/). (I wasn't there and neither have I watched all the videos so take my readings with a grain of salt)

*Update: data Artisans has published an official [recap: On the State of Stream Processing with Apache Flink](https://data-artisans.com/blog/stream-processing-flink-forward-sf-2017-recap)*

### TensorFlow with Flink

> -"What is so hot ?"   
> -"Deep learning"
> 
> -"What is so hot in deep learning ?"    
> -"TensorFlow"

[TensorFlow & Apache Flink](https://www.slideshare.net/FlinkForward/flink-forward-sf-2017-eron-wright-introducing-flink-tensorflow) immediately caught my eye. The basic idea is "TF Graph as a Flink map function" for inference after preprocessing data to off-heap tensor. Online learning is a future direction and the project is [open sourced on GitHub](https://github.com/cookieai/flink-tensorflow/).

### Deep Learning with Flink

Lightbend's Dean Wampler [discussed about how to do Deep Learning with Flink generally](https://www.slideshare.net/FlinkForward/flink-forward-sf-2017-dean-wampler-streaming-deep-learning-scenarios-with-flink), from the challenges in (mini-batch / distributed / online) training and inference to practical approaches, leveraging such Flink features as side inputs and async I/O. He also recommended "Do"s and "Don't"s for the road ahead. One interesting "Don't" is [PMML](https://en.wikipedia.org/wiki/Predictive_Model_Markup_Language).

> PMML - doesn't work well enough. Not really that useful ?

### PMML - not useful ?

At least [ING uses PMML models to bridge the offline training with Spark and online scoring with Flink streaming](https://en.wikipedia.org/wiki/Predictive_Model_Markup_Language). The striking part is how they've decoupled "What" (DAG) and "How" (behavior of each node on the DAG) in the scoring application. Model definition is added at runtime through a broadcast stream without downtime. The same applies for data persist and feature extraction. 

By the way, [Apache Storm has added PMML support in 1.1.0](http://storm.apache.org/2017/03/29/storm110-released.html). 


### It's the data

> -"Which machine learning (ML) framework is the best ?"
> -"All of them"

That's Ted Dunning's answer after learning that his customers typically use 12 ML packages and the smallest number is 5. That's why he didn't talk about Flink ML in [Machine Learning on Flink](https://www.slideshare.net/FlinkForward/flink-forward-sf-2017-ted-dunning-nonflink-machine-learning-on-flink). Even  ML is not the key here. 

> 90%+ of effort is logistics, not learning.

It is the data. Record raw data, use streams to keep data around, and measure and understand (with meta-data) everything. Another thing is to make deployment easier with containerization.

One more lesson for me is there is no such thing as one model. 

> You will have dozens of models, likely hundreds to thousands. 

### Blink Improvements

Blink is Alibaba's fork of Flink. For large scale streaming (> 1000 nodes) at Alibaba, Blink added a bunch of [runtime improvements](https://www.slideshare.net/FlinkForward/flink-forward-sf-2017-feng-wang-zhijiang-wang-runtime-improvements-in-blink-for-large-scale-streaming-at-alibaba). (the right side of "=>" is the problem to solve)

* Native integration with resource management (YARN) => single JobManager for all tasks
* Incremental checkpoint => large state 
* Asynchronous operator => blocking I/O
* Fine-grained recovery from task failures => application restart on one task failure
* Allocation reuse for task recovery => expensive to restore from HDFS
* Non-disruptive JobManager failures via reconciliation => tasks restart on JobManager failure

What's cool is the improvements are being contributed back to the community.

### SQL as building block

Uber shared the [evolution of their business needs and their system evolved accordingly](https://www.slideshare.net/FlinkForward/flink-forward-sf-2017-chinmay-soman-real-time-analytics-in-the-real-world-challenges-and-lessons-at-uber) from event processing, to OLAP, to Streaming SQL on Flink. 

> 70-80% of jobs can be implemented via SQL.

For more on Flink's SQL API, check out [Table & SQL API – unified APIs for batch and stream processing](https://www.slideshare.net/FlinkForward/flink-forward-sf-2017-timo-walther-table-sql-api-unified-apis-for-batch-and-stream-processing) and [Blink's Improvements to Flink SQL And TableAPI](https://www.slideshare.net/FlinkForward/flink-forward-sf-2017-shaoxuan-wangxiaowei-jiang-blinks-improvements-to-flink-sql-and-tableapi)


### To Beam or not to Beam

This is one of Uber's future discussions.   From the [official site](https://beam.apache.org/), 

> Apache Beam provides an advanced unified programming model, allowing you to implement batch and streaming data processing jobs that can run on any execution engine.

Beam has recently added [State and Timer support to unlock new use cases](https://www.slideshare.net/FlinkForward/flink-forward-sf-2017-kenneth-knowles-back-to-sessions-overview) which are portable across runners (e.g. Flink).

### What is Streaming ?

I'd like to wrap up with Stephen Ewen's [answer and high level view of Streaming](https://www.slideshare.net/FlinkForward/flink-forward-sf-2017-stephan-ewen-convergence-of-realtime-analytics-and-datadriven-applications).

> 2016 was the year when streaming technologies became mainstream
> 
> 2017 is the year to realize the full spectrum of streaming applications
> 








