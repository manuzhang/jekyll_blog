--- 
comments: true
layout: post
title: A whirlwind tour of Kafka Summit SF 2018
---

Kafka Summit San Francisco 2018 took place last month with +1200 attendees from +350 companies. [All 60 talks with videos and slides side by side](https://www.confluent.io/resources/kafka-summit-san-francisco-2018/) have been posted on Confluent's website and nicely organized in keynotes and four tracks (Pipeline, Streams, Internals and Business). 

I wasn't there but I've glanced through all the slides and lingered over those I found more interesting. I'd like to share my impressions and provide a whirlwind tour of the conference. Here's an [official wrap-up](https://www.confluent.io/blog/kafka-summit-san-francisco-2018-roundup), by the way.


### Is Kafka a Database 

As you know, Kafka was born a message queue and has grown into a full-fledged streaming platform with [Kafka Connect](https://kafka.apache.org/documentation/#connect), [Kafka Streams](https://kafka.apache.org/documentation/streams/) and [KSQL](https://github.com/confluentinc/ksql). What else can Kafka be ? How about a database ! Martin Kleppmann argues that [Kafka is a database](https://www.confluent.io/kafka-summit-SF18/is-kafka-a-database) and achieves [ACID](https://en.wikipedia.org/wiki/ACID_(computer_science)) properties as in relational databases. This is mind-boggling since relation databases don't provide ACID at scale. Note that you can't set up a Kafka cluster and get ACID for free but need to carefully design your Kafka topics and streaming applications as demonstrated by the author. Still, I have some doubt over whether it can only ensure eventual consistency. 


### Kafka for IoT

* [MQTT](https://en.wikipedia.org/wiki/MQTT) is a lightweight messaging protocol, built for tens of thousands of client connections in unstable network but it lacks in buffering, stream processing, reprocessing and scalability. Those're what Kafka is good at and they make a perfect match to [process IoT data from end to end](https://www.confluent.io/kafka-summit-sf18/processing-iot-data-from-end-to-end) through [Kafka Connect](https://www.slideshare.net/ConfluentInc/processing-iot-data-from-end-to-end-with-mqtt-and-apache-kafka/27), [MQTT Proxy](https://www.slideshare.net/ConfluentInc/processing-iot-data-from-end-to-end-with-mqtt-and-apache-kafka/35) or [REST Proxy](https://www.slideshare.net/ConfluentInc/processing-iot-data-from-end-to-end-with-mqtt-and-apache-kafka/38).

### Global Kafka


* [Booking manages a global Kafka cluster with brokers spread over three zones](https://www.slideshare.net/ConfluentInc/data-streaming-ecosystem-management-at-bookingcom/26). They have containerized replicator, Kafka connect and Kafka monitor on Kubernetes. It' worth mention that they have run into issues with distributed mode of Kafka Connect so the standalone mode is used instead.

* [Linkedin has built its own Brooklin MirrorMaker (BMM)](https://www.slideshare.net/ConfluentInc/more-data-more-problems-scaling-kafkamirroring-pipelines-at-linkedin) to mirror 100+ pipelines across 9 data centers with only 9 BMM clusters. In contrast, Kafka MirrorMaker (KMM) needs 100+ clusters. Moreover, KMM is difficult to operate, poor to isolate failure and unable to catch up with traffic while BMM has dynamic configuration, automatically pausing and resuming mirroring at partition level, and good throughput with almost linear scalability.

* To overcome the explosion of number of KMMs, [Adobe introduces a routing KMM cluster](https://www.slideshare.net/ConfluentInc/beyond-messaging-enterprisescale-multicloud-intelligent-routing/27) to reduce the number of directly connected data center pipelines. They also have written [a nice summary of their Experience Platform Pipeline built on Kafka](https://medium.com/adobetech/creating-the-adobe-experience-platform-pipeline-with-kafka-4f1057a11ef).


### Streaming platform 

When Kafka Streams first came out, I was wondering why I would need another Streaming platform given the place had already been crowded with [Flink](https://flink.apache.org/), [Spark Streaming](https://spark.apache.org/streaming/), [Storm](https://storm.apache.org/), [Gearpump](https://github.com/gearpump/gearpump), etc. Today it strikes me that why I would need another Streaming platform and all the workloads to set up a cluster and maintain when I can do streaming with my Kafka already there. Futhurmore, Confluent adds [KSQL](https://github.com/confluentinc/ksql), a SQL interface, to relieve you of writing cumbersome codes.

* Thus, maybe it's time to [rethink stream processing with Kafka Streams and KSQL](https://www.slideshare.net/ConfluentInc/crossing-the-streams-rethinking-stream-processing-with-kstreams-and-ksql-120334214). There is an illustrative analogy between Kafka ecosystem and Unix Pipeline. ![kafka_analogy](https://image.slidesharecdn.com/05viktorgamov-181022184511/95/crossing-the-streams-rethinking-stream-processing-with-kstreams-and-ksql-12-638.jpg?cb=1540233945)

* Stream joins are not your father's [SQL joins](https://www.w3schools.com/sql/sql_join.asp). How joins are implemented in Kafka Streams and when to use them ? Read this [zen and the art of streaming joins](https://www.slideshare.net/ConfluentInc/zen-and-the-art-of-streaming-joinsthe-what-when-and-why).

* Companies like [Booking](https://www.slideshare.net/ConfluentInc/data-streaming-ecosystem-management-at-bookingcom/5) and [Braze](https://www.slideshare.net/ConfluentInc/realtime-dynamic-data-export-using-the-kafka-ecosystem/33) are building their streaming pipeline around Kafka Connect (data import/export) and Kafka Streams (data processing). ![booking_arch](https://image.slidesharecdn.com/02alexmironov-181023055944/95/data-streaming-ecosystem-management-at-bookingcom-5-638.jpg?cb=1540274422)

* Change Data Capture (CDC) is a way to make use of data and schema changes of your database. [Deberium is a CDC platform for various databases based on Kafka](https://www.slideshare.net/ConfluentInc/change-data-streaming-patterns-for-microservices-with-debezium) from Red Hat. [Pinterest also shares their story of streaming hundreds of TBs of Pins from MySQL to S3](https://www.slideshare.net/ConfluentInc/pinterests-story-of-streaming-hundreds-of-terabytes-of-pins-from-mysql-to-s3hadoop-continuously). 

### Kafka on Cloud

* Intuit have deployed [Kafka on Kubernetes in production](https://www.slideshare.net/ConfluentInc/kafka-on-kubernetesfrom-evaluation-to-production-at-intuit) with proper configurations on load balancing, security, etc.
 
* [Red Hat provide an enterprise distribution of Kafka on Kubernetes/OpenShift](https://www.slideshare.net/ConfluentInc/change-data-streaming-patterns-for-microservices-with-debezium/28) with an open-source upstream, [Strimzi](https://github.com/strimzi). Typical Red Hat. 

* Google is not using Kafka internally but there is a demand for Kafka on Google Cloud Platform (GCP). It's interesting to learn [Google's perspective on Kafka and how they fit Kafka into GCP](https://www.slideshare.net/ConfluentInc/putting-kafka-together-with-the-best-of-google-cloud-platform). 

* Confluent offers some official [recommendations on deploying Kafka Streams with Docker and Kubernetes](https://www.slideshare.net/ConfluentInc/deploying-kafka-streams-applications-with-docker-and-kubernetes/33).

### Internals

As a developer, I love nearly every deck of the Internals track. 

* Zalando shared their [War stories: DIY Kafka](https://www.slideshare.net/ConfluentInc/war-stories-diy-kafka-120353401), especially lessons learned in backing up Zookeeper and Kafka

  > Backups are only backups if you know how to restore them
  
* Another war story from a Kafka veteran, Todd Palino, on [monitoring Linkedin's 5-trillion-message-per-day Kafka cluster](https://www.slideshare.net/ConfluentInc/urp-excuse-you-the-three-metrics-you-have-to-know-120353592). His lesson is not to monitor and alert on everything but to keep an eye on three key metrics, under-replicated partition, request handler and request timing. I totally agree with him that

  > sleep is best in life

* [ZFS makes Kafka faster and cheaper](https://www.slideshare.net/ConfluentInc/kafka-on-zfs-better-living-through-filesystems) through improving cache hit rates and making clever use of I/O devices.

* Jason Gustafson walked you through [failure scenarios of Kafka log replication and hardening it with TLA+ model checker](https://www.slideshare.net/ConfluentInc/hardening-kafka-replication). ZipRecruiter is also [using Chaos Engineering to level up Kafka skills](https://www.slideshare.net/ConfluentInc/using-chaos-engineering-to-level-up-apache-kafka-skills).

* Kafka is famous for using [sendfile system call](http://man7.org/linux/man-pages/man2/sendfile.2.html) to accelerate data transfer. Do you know sendfile can be blocking ? I really enjoy how [Yuto from Line has approached the issue from hypothesis to solution](https://www.slideshare.net/ConfluentInc/kafka-multitenancy160-billion-daily-messages-on-one-shared-cluster-at-line/). (More details at the amazing [KAFKA-7504](https://issues.apache.org/jira/browse/KAFKA-7504)). His [analysis of long GC pause harming broker performance caused by mmap](https://www.slideshare.net/ConfluentInc/one-day-one-data-hub-100-billion-messages-kafka-at-line) from last year is a great computer system lesson as well.

* Who has not secured Kafka ? If you don't know the answer, then Stephane Maarek from DataCumulus [offers some introductions and real-world tips](https://www.slideshare.net/ConfluentInc/kafka-security-101-and-realworld-tips). For example, ![kafka_kerberos](https://image.slidesharecdn.com/02stephanemaarek-181023061947/95/kafka-security-101-and-realworld-tips-18-638.jpg?cb=1540275623)

* [Uber keeps pushing for producer performance improvement without sacrificing at-least-once delivery](https://www.slideshare.net/ConfluentInc/reliable-message-delivery-with-apache-kafka). They shared about their best practices such as increasing replica fetch frequency and reducing thread contention and GC pauses.
 

### Wrap-up

Kafka is definitely the backbone of companies' data architectures and serves as many as billions of messages per day. More and more streaming applications are built with Kafka Connect, Kafka Streams and KSQL. Meanwhile, quite a few companies are managing and mirroring Kafka at global scale. Finally, Kafka on Kubernetes is the new fashion. 


