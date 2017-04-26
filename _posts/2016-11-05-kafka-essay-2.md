---
layout: post
title: "Apache Kafka 三两事（下篇）"
description: "Summarize Apache Kafka Key Experience Items (2)"
category: apache-kafka 
tags: ["apache-kafka"]
---
{% include JB/setup %}
发觉前文内容越写越长了 只能分上下篇了。承接上文，下篇进一步讨论 在Kafka的Producer -> Broker -> Consumer应用层面，如何能最大限度地提高整体Pipeline的吞吐量。

## Consumer Throughput Improvement
### Long Tail Consuming
在消费海量消息时，最理想状况是每个topic parition，publish消息数量一样多 消费速度一样快。这样就能**同时**并行消费完所有的消息了。
下图是公司的某个Kafka集群在5小时内，消费超过2个Topic下的30亿个event。X轴是时间轴，Y轴是单位时间消费event的数量，每个曲线代表一个Topic。可以看到，在最后时段，消费速率极速下降。经过我们进一步分析，发现最后时段某些Topic Partitio早早消费完成，其他还有将近上千万的event在消费中。
**典型的消费不均匀现象**

![Kafka Consumer Throughput]({{ site.JB.IMAGE_PATH }}/kafka_consumer_throughput.png "Kafka Consumer Throughput")
匀速消费，这就对整体的Pipeline 提出2个以下要求:
1. Producer发布消息 要均匀分布到每个Topic Partition，避免单个或某些Partition的热点，积压消息。
2. 对单个Topic而言，Consumer Stream数量要完全均匀（要和Partition数量成正比）。进而对于多个Topic而言，也要成一致性的正比。
细化技术要求而言，需要做到以下几点
1. 消息的发布和消费 能不能乱序处理 
2. Event分配到某个parition的策略能否相对公平
3. 通览全部Topic，Kafka分配Consumer Stream到每台Consumer 实例上
4. Standby Consumer数量是不是足够充分
我并未严格按照每个问题点来逐个回答，因为有些问题之间有依赖和相关性，需要统一解决。
希望下面章节对于回答上述技术难点 能给你些启示。

### Order Matters
Topic Partition是Kafka最小的保持顺序单位。
如果不能支持乱序处理， 对于consumer而言，同一个Key的message就需要推到同一个Topic Partition，就容易产生，某些Partition数据过多  不均匀的问题。
如果支持乱序event 消费（先publish的event后消费），就可以通过简单计数自增 再取模的方式，尽量均匀分摊到每个partition上。
[Default Topic Partition Assignment for each event produce](https://github.com/apache/kafka/blob/trunk/clients/src/main/java/org/apache/kafka/clients/producer/internals/DefaultPartitioner.java#L57-L62)

* Notes: 顺便说一句， 如果对于给定Key 选择对应的parition策略 Kafka实现是通过[MurMur2 Hash 再取模](https://github.com/apache/kafka/blob/0.9.0.0/clients/src/main/java/org/apache/kafka/clients/producer/internals/DefaultPartitioner.java#L83)来实现的。而Murmur hash一直以雪崩效应显著而出名 (参看Appendix#2).

### Idempotent Consumer Behavior
Kafka通过持久化offset方式 尽量保证consume 一次而且仅有一次，但对应consumer 应用而言，更好的方式是如果我们能保证消费消息的后续操作 可以保证幂等。可以大大降低对于offset exactly-once处理要求，例如通过 乐观锁(版本控制) `set value where time_stampe=<given_ts> / version = <given_version>`，或者 确保赋值函数 能满足以下条件 `fn(fn(old_value, delta1),delta2) == fn(fn(old_value, delta2),delta1) `。

### Fair Topic Partition Assignment
在多Topic的场景下，Topic Partition 对应的Consumer Stream分配 容易产生热点。这也是社区[KIP-49 Fair Partition Assignment Strategy](https://cwiki.apache.org/confluence/display/KAFKA/KIP-49+-+Fair+Partition+Assignment+Strategy) 讨论的起源。
Kafka有两种Partition Assignmet策略( [Range_Assignor](https://github.com/apache/kafka/blob/0.9.0.0/clients/src/main/java/org/apache/kafka/clients/consumer/RangeAssignor.java)和 [Round Robin Assignor](https://github.com/apache/kafka/blob/0.9.0.0/clients/src/main/java/org/apache/kafka/clients/consumer/RoundRobinAssignor.java) )。Round Robin策略之前由于严格要求 单个Consumer group内 所有Consumer读取完全相同的Topic List，而无法广泛使用。
下面 引用KIP-49提供例子 展现两大分配策略的分配情况，就能看出分配是否最优了。
假设 我们有以下Topic和对应的Parition数量

| Topic        | Partitions           | 
| ------------- |:-------------:| 
| T1     | 2 |
| T2      | 1      |
| T3 | 2      |
| T4      | 1      |
| T5 | 2      |
然后 我们有4个Consumer 各自消费相应的Topic

| Consumer        | Topics           | 
| ------------- |:-------------:| 
| C1     | T1,T2,T3,T4,T5 |
| C2      | T1, T3, T5  |
| C3 | T1, T3, T5 |
| C4      | T1, T2, T3, T4, T5 |
Range  策略分配结果 

| Consumer        | Topics           | 
| ------------- |:-------------:| 
| C1     | T1-0, T2-0, T3-0, T4-0, T5-0 |
| C2      | T1-1, T3-1, T5-1 |
| C3 |  N/A |
| C4      |  N/A |
Round Robin  策略分配结果

| Consumer        | Topics           | 
| ------------- |:-------------:| 
| C1     | T1-0, T3-0, T5-0 |
| C2      | T1-1, T3-1, T5-1 |
| C3 |  N/A |
| C4      | T2-0,T4-0 |
但是 期待最好的分配结果是 尽量将之后的Parition分配到 已有Partition需要消费的Consumer之外的其他Consumer，尽量分配让每个Consumer都有活可干，避免忙的忙死 闲的闲死。
因此公平的分配方式是类似如下

| Consumer        | Topics           | 
| ------------- |:-------------:| 
| C1     | T2-0, T3-0 |
| C2      | T1-1, T3-1 |
| C3 |  T1-1,T5-0 |
| C4      | T4-0,T5-1 |

* 顺便说一句，现在重看Round Robin策略 社区已经在0.10.2.0版本进行改进 放宽了consumer stream数量限制（所以我们要动态地看待问题哦，社区的发张总是超出我们的想象）。参看[Round-robin partition assignment strategy too restrictive](https://issues.apache.org/jira/browse/KAFKA-2172)

* 为什么Range 策略分配这么不公平呢，这种分配策略优劣在哪呢？
首先，我们先描述一下Range分配策略，看每个单独的Topic，Partition升序排列，对应Consumer Stream按字母顺序升序排列，每个Consumer Stream申请的数量为其步长，然后按顺序分配。不同Topic都是从头开始独立分配，相互没有任何关联性。
 ** 缺点：导致分配结果不公平 没有充分 平衡 利用集群 所有Consumer 的计算和带宽能力
 因而Range 策略就会有如下分配 也就是锁首字母排序靠前的Consumer 只要Available 就一定会优先 更多地分配到Partition, 就有可能 消费的event就会比排名靠后的consumer多很多。
 
| Topic Partition        | Consumer Assignment           | 
| ------------- |:-------------:| 
| T1 P0     | C0 |
| T1 P1      | C2  |
| T2 P0 | C1 |
| T3 P0      | C0 |
| T3 P1      | C1 |
| T4 P0      | C0 |
| T5 P0      | C0 |
| T5 P1      | C1 |

** 优点：分配策略简单 可确定性(deterministic)
为什么分配结果可确定 那么重要呢？这要涉及Consumer Assignment 设计 ( [源码: Client Side Rebalance Partition Assignment](https://github.com/apache/kafka/blob/0.9.0.0/core/src/main/scala/kafka/consumer/ZookeeperConsumerConnector.scala#L682-L758) )。
在0.8-0.9版本里 Kafka Parition Assignment都在Consumer端触发（并非发生在Broker端），因此为了确保最终分配的可验证性，在同样的Global View（包括Topic Partition数目和Consumer注册节点），Kafka需要确保在每个Consumer都能得出统一完全一致的分配结果 [ 分配结果记录每个TopicPartition = 对应 => Consumer Id, Map[TopicAndPartition, ConsumerThreadId] ]。
这样，在下图的balance第5步，即使其他consumer在Zookeeper里提前提交了分配结果（结果必须精确到每个TopicPartition对应给哪个Consumer消费），只有当Consumer之间分配结果一致之后 Balance才能最终完成。否则，Consumer由于分配结果不同 而再次触发Rebalance，直到超过[Rebalance最大限定次数](https://github.com/apache/kafka/blob/0.9.0.0/core/src/main/scala/kafka/consumer/ZookeeperConsumerConnector.scala#L660)。

![Kafka Message Set]({{ site.JB.IMAGE_PATH }}/kafka_consumer_balance_process.png "Kafka Consumer Balance Process")

* Consumer Rebalance & Consumer Redesign
那么 为啥咱不能就Broker Side，由Broker Coordinator来统一进行Consumer Partition Assignment决策 进而通知所有Consumer呢。
社区也有类似的[Kafka 0.9 Consumer Rewrite Design](https://cwiki.apache.org/confluence/display/KAFKA/Kafka+0.9+Consumer+Rewrite+Design) & [Kafka Detailed Consumer Coordinator Design](https://cwiki.apache.org/confluence/display/KAFKA/Kafka+Detailed+Consumer+Coordinator+Design), enable CEntrialized Coordinator，这个design的复杂度在于在考虑到Consumer join-and-leave & Topic Partition add-shrink-expand场景下，如何维护合理的状态机 来保证一致的分配结果。
最新的Proposal：[Kafka Client-side Assignment Proposal](https://cwiki.apache.org/confluence/display/KAFKA/Kafka+Client-side+Assignment+Proposal)

### Standby Consumer
最后 好像提一句 即使所有Consumer消费的Topic Partition足够均匀，也需要多预备些Standby的Consumer，以防止某几个in-running consumer crash之后的再次消费不平衡 导致拖慢整体 End-To-End Throughput。

举例说明 假如我们只有20 Topic Partition，然后对应20 Consumer，分配之后1个Consumer对应一个Topic Partition，消费速度匀速（即使每台机器的SKU配置和JVM应用层各方面配置参数都一致）。
当有一个节点挂了之后，剩下19个节点就会有某一个节点需要consumer 2 Topic Partition，对应的CPU，Memory，network bandwith开销都是和其他节点多一些的，而且单个Kafka Consumer Stream是顺序遍历完一个Parition,再遍历下一个的，消费速度一定和其他节点不一致。

##Ending
关于Topic Partition Assignment策略优化，社区还是有很多in-progress的讨论，具体可以参看 [Appendix3-7]。这里只是抛砖引玉，并未充分展开。
最近 很遗憾没有太多机会去参与Kafka的相关实践。 以后 有机会可以在写以下内容：
* 关于Kafka监控 
* 改进 它如何处理 heartbeat request和data plane request隔离性
* 多少Topic Partition 对应多少连接数的推算
* Kafka面试问题

# Appendix
1. [The Pathologies of Big Data](http://queue.acm.org/detail.cfm?id=1563874)
2. [Murmur Hash](https://research.neustar.biz/2012/02/02/choosing-a-good-hash-function-part-3/)
3. [Kafka Client-side Assignment Proposal](https://cwiki.apache.org/confluence/display/KAFKA/Kafka+Client-side+Assignment+Proposal)
4. [More optimally balanced partition assignment strategy](https://issues.apache.org/jira/browse/KAFKA-2435)
5. [More optimally balanced partition assignment strategy (new consumer)](https://issues.apache.org/jira/browse/KAFKA-3297)
6. [KIP-54 - Sticky Partition Assignment Strategy](https://cwiki.apache.org/confluence/display/KAFKA/KIP-54+-+Sticky+Partition+Assignment+Strategy)
7. [Round-robin partition assignment strategy too restrictive](https://issues.apache.org/jira/browse/KAFKA-2172)

