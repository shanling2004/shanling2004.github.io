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
#### Idempotent Consumer Bahvior
#### Long Tail Consuming

##Kafka Seek API
### Index Structure

##Single & Batch Kafka Message Structure
![Kafka Message Structure]({{ site.JB.IMAGE_PATH }}/kafka_message_format.png "Kafka Message Structure")

| Message   Column      | Description           | Size  |
| ------------- |:-------------:| :---------------:|
| CRC32 CheckSum | 通过CRC32 校验码确认 接收的Payload内容和原先期待的是一致的，否则就fast fail with InvalidMessageException    |    4 Byte |
| Magic | 用于判断消息格式版本号 （在0.9.0Kafkja版本中 暂时看并未完全使用判断）   |    1 Byte |
| Attribue | 该字符可以作为随机的place holder使用, 目前用于表示标识压缩类型 （比如GZIP, SNAPPY, LZ4）   |    1 Byte |
| Key Length |  表示Key的总长度  |    4 Byte |
| Key Payload | Key本身的字符 (可选字符串)  |    K Byte |
| Value Length |  表示Key的总长度  |    4 Byte |
| Value Payload | Value本身的内容   |    V Byte |

![Kafka Message Set]({{ site.JB.IMAGE_PATH }}/messageset.png "Kafka Message Set")


**源码参看**

[Comment for Kafka Message Structure](https://github.com/apache/kafka/blob/0.9.0/core/src/main/scala/kafka/message/Message.scala#L70-L82)

[Byte Buffer Writer to fulifill Kafka Message](https://github.com/apache/kafka/blob/0.9.0/core/src/main/scala/kafka/message/Message.scala#L100-L131)


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

```
The maximum sizes for socket buffers declared via the SO_SNDBUF and
       SO_RCVBUF mechanisms are limited by the values in the
       /proc/sys/net/core/rmem_max and /proc/sys/net/core/wmem_max files.
```
