---
layout: post
title: "Apache Kafka 快问快答"
keywords: [""]
description: ""
category: apache-kafka 
tags: ["apache-kafka"]
---
{% include JB/setup %}

* Why we need kafka comparing with activeMQ rabbitMQ, even zeroMQ？
* What’s the pros & cons at the initial Kafka design? what’s most suitable scenario for Kafka adoption?
* Why we say it’s design for page cache centric? Pros & cons
* How to tuning page cache flush policy? Any impact for data loss?
* Why we say disk write performance is better than in-memory write at some scenario
* What’s the zero-copy meaning in linux IO transmission?
* DO we need really SSD disk for performance enhancement in kafka system as per ROI concern?
* What’s Kafka 0.8 client partition assignment strategy: Range & Radom? Pros & cons. Why they introduce centralized co-ordinator in 0.9? Is 0.8 partition assignment fair ? How to ensure real fair assignment?
* Any benefit or drawback for zookeeper cluster in Kafka cluster?
* Why we need kafka stream library?
* Could you illustrate most of [Kafka Configuration](http://kafka.apache.org/documentation.html#configuration) 's frequent usage?
* How we can guarantee exactly-once pub-sub in kafka wit or without External offset record system. Any benefit when we adopt bloom filter/bit map algorithm?
* What’s the meaning partition? How to balance topic + partition number when designing use case in. 
* What’s ISR? How to coordinate ISR group? Why not use raft algorithm?
* How to trade-off for kafka replica management? Is it too simple? Why not using quorum-based replication? For one topic & one partitionWhat if one publisher send event with ack=1 and anonther publisher send event with ack=all?
* What happen when we encounter “MessageSizeTooLarge” exception?
* How to design consumer & publish side for throughput enlargement or latency shorten requirement. Any key points needed?
* What’s relationship for consumer instances/streams vs Topic partitions. Why we will encounter long-tail consume phenomenon?
* What’s MessageSet mean in Kafka? Any units each kafka message body consist of?
* How to make sure kafka message compression is transparent for consumer app? How trade-off for gzip & snappy compression policy?
* What’s the usage for kafka index? what’s the format for kafka index?
* **Ultimate Question**： if you’re the PMC for new message bus open source project, how to design new Message bus? Any feature in roadmap? Any trade-off? what’s the purpose for new one?