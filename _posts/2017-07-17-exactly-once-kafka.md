---
comments: true
layout: post
title: Exactly-once in Kafka
--- 

[Exactly-once finally landed in Kafka 0.11](https://www.confluent.io/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/) with idempotent producer per partition and atomic writes across multiple partitions. Furthermore,

> Building on idempotency and atomicity, exactly-once stream processing is now possible through the Streams API in Apache Kafka

We also get to know Kafka team has gone through a [meticulous design](https://goo.gl/fnycgk)(> 60 pages, man), iterative development process and extensive tests to ensure correctness and low performance overhead of Exactly-once guarantee. 

Lastly, it's warned that Exactly-once is not Magical Pixie Dust you can sprinkle on your application.

> **Exactly-once processing is an end-to-end guarantee** and the application has to be designed to not violate the property as well. If you are using the consumer API, this means ensuring that you commit changes to your application state concordant with your offsets as described here.
> 

This has stirred up heated discussions (on [Reddit](https://www.reddit.com/r/programming/comments/6kh65f/exactlyonce_semantics_is_possible_heres_how/) and [HackerNews](https://news.ycombinator.com/item?id=14670801)) where some people are skeptical whether Exactly-once is (mathematically) possible linking to consensus problems like [Two General Problem](https://en.wikipedia.org/wiki/Two_Generals%27_Problem).

Several distributed systems veterans ([credit to Stephan Ewen](https://twitter.com/StephanEwen/status/882298788827869184)) have weighed in to sort out the confusion for us. 

* Kafka co-creator, Jay Kreps [responded on HackerNews](https://news.ycombinator.com/item?id=14671305) and [explained in depth](https://medium.com/@jaykreps/exactly-once-support-in-apache-kafka-55e1fdd0a35f) how Exactly-once is possible and how Kafka has supported it. (I'd like to get back to some of Jay's words later)

* Zookeeper and Bookeeper PMC, Flavio Junqueira argued that there is [No consensus in exactly-once](https://fpj.me/2017/07/04/no-consensus-in-exactly-once/).

  > I have argued here that exactly-once delivery and consensus are not equivalent, and in fact, suggested that it is a weaker problem compared to consensus primarily because it does not require order of delivery.
  
  and what Exactly-once means in practice.
  
  > Exactly-once intuitively means that something happens once and only once. In the context of some current systems, however, the term implies that something that happens multiple times is **effective only once**. Multiple applications of a transformation or message delivery only affect the state of a given application or system once.
  
* In [You Cannot Have Exactly-Once Delivery Redux](http://bravenewgeek.com/you-cannot-have-exactly-once-delivery-redux/), Tyler Treat reminded us of the difference between "Exactly-once delivery" and "Exactly-once processing".
  
  > “Delivery” is a transport semantic. “Processing” is an application semantic.
  
  and the latter is possible in a closed system which Kafka provides.
  
  > To achieve **exactly-once processing semantics**, we must have a closed system with end-to-end support for modeling input, output, and processor state as a single, atomic operation. Kafka supports this by providing a new transaction API and idempotent producers. 
 
I also looked into the fault tolerance semantics in [Flink Streaming](https://ci.apache.org/projects/flink/flink-docs-release-1.3/dev/connectors/guarantees.html) and [Spark Structured Streaming](http://spark.apache.org/docs/latest/structured-streaming-programming-guide.html#fault-tolerance-semantics), both of which requires coordination of source, sink and the checkpointing/snapshoting mechanism of the system.

* Flink Streaming (v1.3)

  > Flink can guarantee **exactly-once state updates** to user-defined state only when the source participates in the snapshotting mechanism.

  > To guarantee **end-to-end exactly-once record delivery** (in addition to exactly-once state semantics), the data sink needs to take part in the checkpointing mechanism.

* Spark Structured Streaming (v2.2.0)

  > The engine uses checkpointing and write ahead logs to record the offset range of the data being processed in each trigger. 
  
  > Together, using replayable sources and idempotent sinks, Structured Streaming can ensure **end-to-end exactly-once semantics** under any failure

Exactly-once delivery, Exactly-once processing, Exactly-once semantics or Effectively once, we have seen all usage of these words to mean the same thing. What on earth is Exactly-once ? Jay Kreps said it best,

> the real guarantee people want is the **end-to-end correct processing** of messages in the presence of failure without having to think hard about the integration with their app.

Let's use "end-to-end correct processing" from now on. I like it more what Jay concluded his article with 

> Rather than giving up and punting all the hard problems onto the poor person implementing the application we should strive to understand how we redefine the problem space to build correct, fast, and most of all **usable system primitives** they can build on

