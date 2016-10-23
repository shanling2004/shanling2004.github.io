---
layout: post
title: "Apache Kafka 三两事"
description: "Summarize Apache Kafka Key Experience Items"
category: apache-kafka 
tags: ["apache-kafka"]
---
{% include JB/setup %}
今年多少做了些Apache Kafka相关的项目，看了些源码和很多社区的分享( 主要是[linkedin] (https://engineering.linkedin.com/) 和 [confluent.io](http://www.confluent.io/blog/) ), 这里多少做个总结, 留给未来的自己回顾，朝花夕拾。

**Note**: 本文大部分是基于Apache Kafka 0.8.2和0.9.0版本讨论的。
				
##高性能
首先，还是想老生常谈 说一下自己对于Kafka高性能表现的理解。
High Performance和High Throughput是很多分布式系统都追求和标榜的。Apache Kafka在某些方面（JVM 内存开销，磁盘写操作，数据包的transfer）做到了一定的极致。
他们是如何让牛皮变成现实的？下面，再次回顾一下几个关键点。

###Faceless Event & Direct Buffer & Page Cache
为了尽可能避免GC Pause的影响, Apache Kafka Broker接收到消息之后 并没有把消息负载 加载到JVM Heap中,而是直接加载到Direct Buffer里，进而直接转入Page Cache，再通过Page Cache Asynchronous和底层存储介质打交道 完成Event Flush.

###WAL
###SendFIle API’s Zero Copy
####Batch EveryWhere
### Async Process

这章节最后，我想说 有得必有失，在追求某方面极致的过程中 必定在其他方面有所缺失 或者照顾不周。

##性能调优
### OS Layer
### Topic Partition Number
### Producer & Consumer Settings
#### Order Matters
#### Idempotent Consumer Bahvior


Kafka Seek API
##Kafka Event Structure
![Kafka Message Structure]({{ site.JB.IMAGE_PATH }}/kafka_message_format.png "Kafka Message Structure")

| Message   Column      | Description           | Size  |
| ------------- |:-------------:| :---------------:|
| CRC32 CheckSum | 通过CRC32 校验码确认 接收的Payload内容和原先期待的是一致的，否则就fast fail with InvalidMessageException    |    4 Byte |
| CRC32 CheckSum | 通过CRC32 校验码确认 接收的Payload内容和原先期待的是一致的，否则就fast fail with InvalidMessageException    |    4 Byte |

[Comment for Kafka Message Structure](https://github.com/apache/kafka/blob/0.9.0/core/src/main/scala/kafka/message/Message.scala#L70-L82)
[Byte Buffer Writer to fulifill Kafka Message](https://github.com/apache/kafka/blob/0.9.0/core/src/main/scala/kafka/message/Message.scala#L100-L131)

## Index Structure
## Fair Topic Partition Assignment
## Consumer Rebalance & Consumer Redesign
