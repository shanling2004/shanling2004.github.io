---
layout: post
title: "Apache Kafka 三两事（下篇）"
description: "Summarize Apache Kafka Key Experience Items (2)"
category: apache-kafka 
tags: ["apache-kafka"]
---
{% include JB/setup %}
发觉前文内容越写越长了 只能分上下篇了。下篇只在更深入地理解Kafka 某些方面的特性 内部结构和一些设计建议。
## Consumer Throughput Improvement
#### Order Matters
Topic Partition是Kafka最小的保持顺序单位。
如果不能支持乱序处理， 对于consumer而言，同一个Key的message就需要推到同一个Topic Partition，就容易产生，某些Partition数据过多  不均匀的问题。
如果支持乱序event 消费（先publish的event后消费），就可以通过简单计数自增 再取模的方式，尽量均匀摊到每个partition上。
[Default Topic Partition Assignment for each event produce](https://github.com/apache/kafka/blob/trunk/clients/src/main/java/org/apache/kafka/clients/producer/internals/DefaultPartitioner.java#L57-L62)

* Notes: 顺便说一句， 如果对于给定Key 选择对应的parition策略 Kafka实现是通过MurMur2 Hash 再取模来实现的。而Murmur hash一直以雪崩效应显著而出名（参看Appendix#6）

#### Idempotent Consumer Bahvior
Kafka通过持久化offset方式 尽量保证consume 一次而且仅有一次，但更好的方式是如果我们能保证消费消息的后续操作 可以保证幂等。可以大大降低对于offset exactly-once处理要求。
#### Long Tail Consuming
消费海量消息时，最理想状况是每个topic parition，publish消息数量一样多 消费速度一样快。这样就能**同时**并行消费完所有的消息了。
![Kafka Consumer Throughput]({{ site.JB.IMAGE_PATH }}/kafka_consumer_throughput.png "Kafka Consumer Throughput")



## Fair Topic Partition Assignment
## Consumer Rebalance & Consumer Redesign
### Monitor
Broker Controller
Request in queue & Request handler thread number & request isolation

Partition Leader

# Appendix
1. [The Pathologies of Big Data](http://queue.acm.org/detail.cfm?id=1563874)
2. [TCP State/Transition Diagram](https://tangentsoft.net/wskfaq/articles/debugging-tcp.html)
3. [TCP Window Scale](https://slaptijack.com/system-administration/what-is-tcp-window-scaling/)
4. [Kafka Compression Gzip vs Snappy ](https://nehanarkhede.com/2013/03/28/compression-in-kafka-gzip-or-snappy/)
5. [TCP Man Page](http://man7.org/linux/man-pages/man7/tcp.7.html)
6. [Murmur Hash](https://research.neustar.biz/2012/02/02/choosing-a-good-hash-function-part-3/)


