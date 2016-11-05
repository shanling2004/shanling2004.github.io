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
为了尽可能避免GC Pause的影响 [ 尤其在是高并发的时候 对于99percentile performance metrics的影响 ], Apache Kafka Broker接收到消息之后 并没有把消息负载 长期加载在JVM Heap中, 而是批量留驻到Direct Buffer和Mapped ByteBuffer里, 进而批量flush到Page Cache, 再通过Page Cache Asynchronous和底层存储介质打交道 完成MessageSet Flush。

* **Faceless Event Handle**

需要注意的是 整个过程中Kafka Broker并没有对Message做任何反序列化的操作。从ProducerRequest进入 handle，到最后Index和log segment保存 操作对象都是 [ByteBufferMessageSet.scala](https://github.com/apache/kafka/blob/trunk/core/src/main/scala/kafka/message/ByteBufferMessageSet.scala), 不涉及任何具体的POJO entity转换。

* **Direct Buffer (off-heap)**

**流程图如下**， 其中segment＃append 把 Message Byte Array常驻在Kafka Log 映射的FileChannel，以及更新 Kafka Index对应的Mapped ByteBuffer。这些数据结构都是off-heap的, 不会触发GC, 反之GC Pause也会阻止他们的操作。
另外,为了能更好地支持高吞吐量, Segment flush to disk 也是定时flush的, 并非on-demand 触发。

* **Page Cache & PDflush**

![Kafka Producer Request Handle Sequence Diagram In Broker]({{ site.JB.IMAGE_PATH }}/kafka_producer_req_handle.png "Kafka Producer Request Handle Sequence Diagram In Broker")

如果结合Page Cache Pdflush机制，完整地看待Kafka 接收Prodcuer request到message 保存到磁盘介质的话，请参看以下流程图：

![Kafka Page Cache Disk Flush]({{ site.JB.IMAGE_PATH }}/page_cache_flush.png "Kafka Page Cache Disk Flush")

最后 我们看个实际例子来确认kafka基于Page Cache为中心的消息存储思想, 下图展示我们产品环境的内存开销，当前系统 没有额外高负载进程 除了Kafka broker，可以看到Kafka Broker最多也就占用了13GB，整个Page Cache占用了超过110GB的memory footprint. 

![Kafka Broker Memory Consumption]({{ site.JB.IMAGE_PATH }}/memory_consumption.png "Kafka Broker Memory Consumption")

###WAL
Write-Ahead log flush主要还是想充分利用性能友好的磁盘顺序写。为了最大化提升读写性能，Kafka 追加event 在[log segment](https://github.com/apache/kafka/blob/trunk/core/src/main/scala/kafka/log/LogSegment.scala#L97) 和 [index](https://github.com/apache/kafka/blob/trunk/core/src/main/scala/kafka/log/LogSegment.scala#L105) byte buffer中 定期flush到page cache，进而刷到磁盘。

引用之前经典的 关于随机，顺序磁盘访问和内存访问的性能评测。需要强调的是磁盘随机读相比于磁盘顺序读慢了将近150，000倍，甚至于内存随机读性能也劣于磁盘顺序。

![Comparing Random and Sequential Access in DIsk and Memory]({{ site.JB.IMAGE_PATH }}/jacobs3.jpg "Comparing Random and Sequential Access in DIsk and Memory")

但是 有一点我不太明白的是为什么顺序读 SAS磁盘 ( 53.2M values/sec ) 会优于SSD ( 42.2M values/sec )。

###SendFile API’s Zero Copy
大多数场景下，磁盘数据读取, 进而通过网络传输到远端的服务器上。

整个过程中，Kernel从磁盘读取数据, 在推到用户态的程序内存中, 然后再从用户态反推回Kenerl态，在通过socket buffer网络传输出去。其中，第二第三步骤显得多余而低效. 既浪费了CPU时钟资源，内存资源，同时两次用户态和Kernel态的切换，是相对比较昂贵的System Call **Trap Interrupt**操作, 涉及到上下文的切换. 

![Comparing Random and Sequential Access in DIsk and Memory]({{ site.JB.IMAGE_PATH }}/sendfile_2.gif "Comparing Random and Sequential Access in DIsk and Memory")

这就催生了ZeroCopy需求的OS API 支持, 通过Kenerl态内部Read Buffer和Socket Buffer拷贝，就完成网络传输的数据准备，完全不依赖用户态的任何操作和开销。

![Zero Copy]({{ site.JB.IMAGE_PATH }}/sendfile.gif "Zero Copy")

Kafka broker利用[ FileChannel#transferTo API ](https://github.com/apache/kafka/blob/0.9.0.0/core/src/main/scala/kafka/log/FileMessageSet.scala#L165)来调用底层操作系统的[SendFile函数](https://github.com/torvalds/linux/blob/master/fs/read_write.c#L1400-L1402), 使得所有incoming log追加都是Zero Copy, 省时省力。

####Batch EveryWhere
无论是Producer batch flush还是Consumer batch consume和Broker本地log segment保存MessageSet, Kafka无时无处都体现batch events的概念，批量地处理event inbound outbound和存储。这为之后throughput tuning提供了基础构架的支持。

![Kafka Message Set]({{ site.JB.IMAGE_PATH }}/batch_process.png "Kafka Message Set")


这章节最后，我想说 有得必有失，在追求某方面极致的过程中 必定在其他方面有所缺失和取舍。

##性能调优
### OS Layer
#### TCP kernel parameters optimization 
```
net.ipv4.tcp_fin_timeout = 30net.ipv4.tcp_keepalive_time = 360net.ipv4.tcp_sack = 1net.ipv4.tcp_dsack = 1net.ipv4.tcp_timestamps = 1net.ipv4.tcp_window_scaling = 1net.ipv4.tcp_tw_reuse = 1net.ipv4.tcp_tw_recycle = 1net.core.wmem_max = 8388608net.core.rmem_max = 8388608net.ipv4.tcp_rmem = 4096        87380   8388608net.ipv4.tcp_wmem = 4096        87380   8388608net.ipv4.ip_local_port_range = 1024 65000net.ipv4.tcp_max_tw_buckets = 5000net.core.netdev_max_backlog = 262144net.core.somaxconn = 262144
```
#### Page Cache
#### FD Number

### Topic Partition Number
### Producer & Consumer Settings
总体思路如下：
* 在SLA能接受的情况下，批量分发Kafka Event 以期较高的吞吐量
* 相比于单批次的Kafka Batch Event数据量，尽量确保每个TCP round trip都能发送足够多的data，以保证用尽量少的TCP/socket 发送接收的round trip
* 用两端的CPU Cycle做event payload压缩和解压缩 换取精简的event payload，以减少对network bandwidth和broker storage的压力和要求
* 平衡Low Latency和data replica to prevent data loss的需求，在大多数场景下，Kafka Partition Leader已经接收消息 确认之后 消息通常就不太会丢失 通过offline replica sync 异步地分发到其他follower节点上。 
#### Producer Settings
```
/** The producer will attempt to batch records together into fewer requests whenever multiple records are being sent **/
batch.size=1048576
send.buffer.bytes=1048576
ack=1
retries=3
buffer.memory=104857600
linger.ms=300
/* when buffer is full, then new incoming message is blocked till the buffer get released */
block.on.buffer.full=true	
/* to save network bandwidth, each event is GZIP compressed which try to use more CPU cycle used for compression to save network bandwidth */
compression.type=snappy 
```

* 关于"batch.size"配置, 请参看[RecordAccumulator#append 方法](https://github.com/apache/kafka/blob/0.9.0/clients/src/main/java/org/apache/kafka/clients/producer/internals/RecordAccumulator.java#L178-L197)，简单来说Producer Record Accumulator预先分配一个MemoryRecords, 预分配off-heap byteBuffer, 大小受"batch.size" 控制, 所有新来的event都通过`public FutureRecordMetadata tryAppend(byte[] key, byte[] value, Callback callback)` API 来加入到Memory Record. 
* 关于"send.buffer.bytes"配置，请参看[NetworkClient#initiateConnect 方法](https://github.com/apache/kafka/blob/0.9.0/clients/src/main/java/org/apache/kafka/clients/NetworkClient.java#L489-L492)，简单来说就是创建socket connection with 配置的sender buffer和receive buffer size.

因此 最理想情况下 如果我们在一次socket 发送 进而单次TCP 数据发送就完成一个批次的Kafka event,效率是最高的, 所以需要满足以下的配置条件:
```
batch.size <= send.buffer.bytes <= net.core.wmem_max
```

再不济，需要满足`batch.size <= N * send.buffer.bytes <= M * net.core.wmem_max`, 以满足数据对齐发送的效果, 珍惜每个TCP和socket发送的窗口。

* 关于压缩策略，比较Snappy和Gzip优劣，一般场景下，从吞吐量和CPU资源消耗角度看，Snappy都优于Gzip，除非是特别敏感于传输数量大小，GZIP是更优秀的选择（因为GZIP压缩比更高），例如跨数据中心的传输。参看附录#4

| Comparison Item      | Snappy           | Gzip  |
| ------------- |:-------------:| :---------------:|
| Compression Ratio | 2:1    |    2。8:1 |
| Throughput Ratio | 2.5X   |    1X |
| CPU Cycle Usage | Less  |   More |
#### Consumer Settings
```
queued.max.message.chunks=10
fetch.message.max.bytes=1048576
socket.receive.buffer.bytes=1048576
```
![Kafka Message Set]({{ site.JB.IMAGE_PATH }}/kafka consumer.png "Kafka Message Set")

* Note:
[based on Fetch SIze，prepare fetch request to pull partition data](https://github.com/apache/kafka/blob/0.9.0.0/core/src/main/scala/kafka/consumer/ConsumerFetcherThread.scala#L44)
[Fetch Request's Partition Data append into blocking queue](https://github.com/apache/kafka/blob/0.9.0.0/core/src/main/scala/kafka/consumer/ConsumerFetcherThread.scala#L74)

* 未完，下篇接着扯。

# Appendix
1. [The Pathologies of Big Data](http://queue.acm.org/detail.cfm?id=1563874)
2. [TCP State/Transition Diagram](https://tangentsoft.net/wskfaq/articles/debugging-tcp.html)
3. [TCP Window Scale](https://slaptijack.com/system-administration/what-is-tcp-window-scaling/)
4. [Kafka Compression Gzip vs Snappy ](https://nehanarkhede.com/2013/03/28/compression-in-kafka-gzip-or-snappy/)
5. [Producer Performance Tuning For Apache Kafka](http://www.slideshare.net/JiangjieQin/producer-performance-tuning-for-apache-kafka-63147600) 
6. [TCP Man Page](http://man7.org/linux/man-pages/man7/tcp.7.html)

```
The maximum sizes for socket buffers declared via the SO_SNDBUF and
       SO_RCVBUF mechanisms are limited by the values in the
       /proc/sys/net/core/rmem_max and /proc/sys/net/core/wmem_max files.
```
