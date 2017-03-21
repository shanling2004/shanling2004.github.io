---
layout: post
title: "Apache Kafka 三两事 (上篇)"
description: "Summarize Apache Kafka Key Experience Items"
category: apache-kafka 
tags: ["apache-kafka"]
---
{% include JB/setup %}

#Release Notes
| Version      | Comment       |
| ------------- |:---------------:|
| 1 | Init Commit |
| 2 | 1. 删除 disable drop cache对data loss的影响 章节 感谢迪八哥和XuJian的指正。 2. 增加 Multi data Volume Support章节 3.增加Appendix#11  |

今年多少做了些Apache Kafka相关的项目，看了些源码和很多社区的分享( 主要是[linkedin] (https://engineering.linkedin.com/) 和 [confluent.io](http://www.confluent.io/blog/) ), 这里多少做个总结, 留给未来的自己回顾，朝花夕拾。本文主要想探究一下从设计角度来看 Kafka高性能和高吞吐量的秘密，进而如何有针对性的tuning来达到峰值的吞吐量。

**Note**: 本文大部分是基于Apache Kafka 0.9.0版本讨论的。操作系统内核是基于Linux Kenerl 2.6+ 版本。
				
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

###WAL & Sequence IO
Write-Ahead log flush主要还是想充分利用性能友好的磁盘顺序写。为了最大化提升读写性能，Kafka 追加event 在[log segment](https://github.com/apache/kafka/blob/trunk/core/src/main/scala/kafka/log/LogSegment.scala#L97) 和 [index](https://github.com/apache/kafka/blob/trunk/core/src/main/scala/kafka/log/LogSegment.scala#L105) byte buffer中 定期flush到page cache，进而刷到磁盘。

引用之前经典的 关于随机，顺序磁盘访问和内存访问的性能评测。需要强调的是磁盘随机读相比于磁盘顺序读慢了将近150，000倍，甚至于内存随机读性能也劣于磁盘顺序。

![Comparing Random and Sequential Access in DIsk and Memory]({{ site.JB.IMAGE_PATH }}/jacobs3.jpg "Comparing Random and Sequential Access in DIsk and Memory")
原文链接参看Appendix#11

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
#### TCP Optimization 
TCP tuning主要思路是尽可能扩大滑动窗口的大小，能迅速到达窗口峰值，尽可能保留每个已经建立的TCP连接（毕竟TCP三次握手建立连接很费时））
```
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_keepalive_time = 360
net.ipv4.tcp_sack = 1
net.ipv4.tcp_dsack = 1
net.ipv4.tcp_timestamps = 1
net.ipv4.tcp_window_scaling = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_tw_recycle = 1
net.core.wmem_max = 8388608
net.core.rmem_max = 8388608
net.ipv4.tcp_rmem = 4096        87380   8388608
net.ipv4.tcp_wmem = 4096        87380   8388608
net.ipv4.ip_local_port_range = 1024 65000
net.ipv4.tcp_max_tw_buckets = 5000
net.core.netdev_max_backlog = 262144
net.core.somaxconn = 262144
```
* 以上参数值仅供参考
#### Page Cache
说实话 Page cache tuning 是Kafkatuning比较重要的部分 毕竟之前的阐述表明Kafka 设计是Page cache centric，银耳作为基础核心依赖的部分tuning 做到位的话 效果也是事半功倍的。
##### Disable Swappniess
为了达到最后的cache 效果，我们不想利用磁盘SWAP分区来补充内存空间，最大限度利用物理内存空间。
```
echo 0> /proc/sys/vm/swappiness
```

##### Page Cache Settings
```
vm.dirty_expire_centisecs
vm.dirty_writeback_centisecs 
vm.dirty_ratio
vm.dirty_background_ratio
vm.nr_pdflush_threads
```
vm.dirty_background_ratio 是内存可以填充“脏数据”的百分比。这些“脏数据”在稍后是会写入磁盘的，pdflush/flush/kdmflush这些后台进程会稍后清理脏数据。

vm.dirty_ratio 是绝对的脏数据限制，内存里的脏数据百分比不能超过这个值。如果脏数据超过这个数量，新的IO请求将会被阻挡，直到脏数据被写进磁盘。这是造成IO卡顿的重要原因，但这也是保证内存中不会存在过量脏数据的保护机制。

vm.dirty_expire_centisecs 指定脏数据能存活的时间。在这里它的值是30秒。当 pdflush/flush/kdmflush 进行起来时，它会检查是否有数据超过这个时限，如果有则会把它异步地写到磁盘中。毕竟数据在内存里待太久也会有丢失风险。单位是厘秒(1/100 second)。

vm.dirty_writeback_centisecs 指定多长时间 pdflush/flush/kdmflush 这些进程会起来一次。单位是厘秒(1/100 second)。
nr_pdflush_threads 指定多少线程帮助并发的flush page cache脏数据到磁盘。

* 建议
  * vm.dirty_background_ratio < vm.dirty_ratio, 在内存空间充足的场景下，可以适当调大比例，防止IO block
  * vm.dirty_writeback_centisecs flush频度 调太大容易导致过多dirty page cache，太频繁，容易导致不必要的小数据量读写IO，要视情况而定，据说1:6 (dirty_expire_centisecs  :    dirty_writeback_centisecs )的比例比较好，但我并未测试证明过。
  * nr_pdflush_threads根据实际情况可以适当调大以满足快速flush需求
  * 关于如何检测in-running dirty page和当前wirte back数量，可以简单利用以下命令

```
shell:/proc/sys/vm# cat /proc/vmstat | egrep "dirty|writeback"
nr_dirty 61
nr_writeback 0
```

#### File Descriptor Number
单个Topic Parition的commit log，会每个segment对应两个file descriptor，一个对应*.log原生event文件，另一个FD对应 *.index索引文件。
因而，我们可以大致推导出单topic所需要的FD数量，如下:

```
FD_number_per_topic=
Topic_number * Partition_number * segment_number * replica_ratio / broker_node_number
```
另一方面，`log.segment.bytes`单个segment的log最大byte数目和单topic parition的log byte总量都会影响open FD总量。

* 控制FD 数量,可以通过三种方式: (1)`ulimit command` (2)`/etc/security/limits.conf` (3)`/proc/sys/fs/file-max`

```
shell:/x/home/zhiling# ulimit -Hn
51200
shell:/x/home/zhiling# ulimit -Sn
51200

shell:# cat  /etc/security/limits.conf|grep nofile
***       soft    nofile          51200
***       hard    nofile          51200

shell:/x/home/zhiling# cat /proc/sys/fs/file-max
262144
```
### Kafka App Layer
##### Topic Partition Number
Kafka Partition 是Kafka最小的并发单位，更多的Partition意味着有更多的独立通道可以rang生产消费者两端传输消息。因此，Partition number很大程度上决定了Kafka 的消息吞吐量。
单Partition内部event是保证顺序的，跨parition间的event是不保证顺序。因而，如果你对某组event希望保持顺序地消费和发生，需要好好定义Event Key。
##### Partition增加的副作用
月满则亏，partition数量也不能无限制地扩大 追求无限的吞吐量。有哪些因素限制Partition number增长呢。
* 更多的分区会提供更大吞吐量
* 更多的分区需要打开更多的文件句柄
* 过多分区可能会影响可用性
* 更多分区会增加端到端延迟
* 更多分区会要求客户端更多内存分配
具体，请参看Appendix#10

###### 维度划分
之所以，会突兀地提出这个话题，主要之前做项目时候有个tradeoff和决定，是在保持较高的吞吐量的情况下，关于如何平衡业务关联的Topic Partition和Kafka cluster运维。
举个例子，我们有*300* Topic 关于各种不同的业务含义事件，为了保持一定并发度，假设我们给每个Topic分配*20*个Partition。整体，单个Cluster就需要支持6000 Topic+Partition。
随着业务需求的增长，持续会有更多的Topic加入，过多的Topic+Partition就会引发之前提到的副作用，对整体集群的维护增加更多的负担，ATB也会受影响。
解决方案有两个 （1）[ Scale on Separate Kafka Cluster ] 引入新Kafka集群，在Cluster层面去扩展 （2）[ Virtual Topic Share Physical Topic ] 把多个业务Topic 整合成单个大的Topic，例如 和用户相关的Virtual Topic Event可以可以生产消费在同一个物理的Topic上。

| Comparison Item      | Pros           | Cons  |
| ------------- |:-------------:| :---------------:|
| Scale on Separate Kafka Cluster | 细粒度，对于consumer和MirrorMaker相对友好 （可以有个相应的Topic Pattern归类一组有类似业务含义的Topic Group） | 可能会有更多硬件投入，需要有smart consumer & publisher 封装不同Cluster之间的差异，能自动映射Topic -> Cluster 关系 |
| Virtual Topic Share Physical Topic  |    共用大Partition数量的Topic，为每个Virtual Topic增加共用的并行度 |    粗粒度 很难拆分 对于consumer而言，如果某个consumer只想，或者MirrorMaker想对每个不同的Virtual Topic做各种不同的Replica 策略的话 就只能filter其他Virtual Topic的event了 |

所以，要如何规划Kafka集群，以下是我的理解。
大多数场景 我推荐方案1[ Scale on Separate Kafka Cluster ] 。
* Topic: 定义某组有业务含义的归类（例如，用户交易事件，用户登录事件，用户退出事件），mirrorMaker也可以轻松地根据topic名做cross colo replica。
* Partition: 内部调整吞吐量的参数 不关联任何业务含义
* Cluster：只有当单个集群 无法支撑更多topic partition traffic的时候，我们可以扩展独立的新集群来容纳新的业务含义的完整topic。

##### Multi Disk Volume Support
最坏情况下，如果Page Cache一直miss match，不得不从commit log的磁盘上读取event。在Kafka应用层面，可选优化方式是为commit log目录指定多个磁盘卷。每个磁盘卷可以挂载各自独立的磁盘，因而即使是机械磁盘，他们之间的磁道寻址也是相互独立 可并行执行的。
例如，如下配置绑定三个目录
```
logs.dir=/x/kafka/data01/kafka-app-logs,/x/kafka/data02/kafka-app-logs,/x/kafka/data03/kafka-app-logs
```
那么 有个问题是到底Kafka内部怎么决定每个Partition commit和index log放在哪个folder呢？可以自定义配置吗？
答案是：每次都挑选Parititon数量最少的目录作为下一个创建新Partition的目录，暂时无法定制化。

个人觉得这个并不完全准确，因为每个Partition的event数量会有所不同，对应的segment数量就不同，每个segment才会最终对应物理的commit log和index log文件。

[查找下一个Log Dir的源码逻辑](https://github.com/apache/kafka/blob/trunk/core/src/main/scala/kafka/log/LogManager.scala#L399-L417)

这里不涉及存储内部优化 [E.g. 磁盘介质选择（SSD vs SATA）,或者RAID阵列优化或者分布式存储系统)

#### Producer & Consumer Settings
总体思路如下：
* 在SLA能接受的情况下，批量分发Kafka Event 以期较高的吞吐量
* 相比于单批次的Kafka Batch Event数据量，尽量确保每个TCP round trip都能发送足够多的data，以保证用尽量少的TCP/socket 发送接收的round trip
* 用两端的CPU Cycle做event payload压缩和解压缩 换取精简的event payload，以减少对network bandwidth和broker storage的压力和要求
* 平衡Low Latency和data replica to prevent data loss的需求，在大多数场景下，Kafka Partition Leader已经接收消息 确认之后 消息通常就不太会丢失 通过offline replica sync 异步地分发到其他follower节点上。 
##### Producer Settings
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
| Compression Ratio | 2:1    |    2.8:1 |
| Throughput Ratio | 2.5X   |    1X |
| CPU Cycle Usage | Less  |   More |
##### Consumer Settings
```
queued.max.message.chunks=10
fetch.message.max.bytes=1048576
socket.receive.buffer.bytes=1048576
```
![Kafka Message Set]({{ site.JB.IMAGE_PATH }}/kafka consumer.png "Kafka Message Set")

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


* Note:
[based on Fetch Size，prepare fetch request to pull partition data](https://github.com/apache/kafka/blob/0.9.0.0/core/src/main/scala/kafka/consumer/ConsumerFetcherThread.scala#L44)
[Fetch Request's Partition Data append into blocking queue](https://github.com/apache/kafka/blob/0.9.0.0/core/src/main/scala/kafka/consumer/ConsumerFetcherThread.scala#L74)

* 未完，下篇接着扯。最后 我想说在文章最后 才发现Kafka Committer (Jay Kreps, 也是最近比较红的一篇文章"The Log: What every software engineer should know about real-time data's unifying abstraction"作者)，他列举的三大理由（参看Appendix#8） 本文基本都有所涉及 当然，阐述的深度和深入浅出肯定和大神不是一个级别的。

# Appendix
1. [The Pathologies of Big Data](http://queue.acm.org/detail.cfm?id=1563874)
2. [TCP State/Transition Diagram](https://tangentsoft.net/wskfaq/articles/debugging-tcp.html)
3. [TCP Window Scale](https://slaptijack.com/system-administration/what-is-tcp-window-scaling/)
4. [Kafka Compression Gzip vs Snappy ](https://nehanarkhede.com/2013/03/28/compression-in-kafka-gzip-or-snappy/)
5. [Producer Performance Tuning For Apache Kafka](http://www.slideshare.net/JiangjieQin/producer-performance-tuning-for-apache-kafka-63147600) 
6. [TCP Man Page](http://man7.org/linux/man-pages/man7/tcp.7.html)
7. [Linux Storage Stack Daigram](https://upload.wikimedia.org/wikipedia/commons/3/30/IO_stack_of_the_Linux_kernel.svg)
8. [Why kafka Performance rocks](https://www.quora.com/Kafka-writes-every-message-to-broker-disk-Still-performance-wise-it-is-better-than-some-of-the-in-memory-message-storing-message-queues-Why-is-that)
9. [The Log: What every software engineer should know about real-time data's unifying abstraction](https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying)
10. [翻译:在Kafka集群内, 如何权衡Topics／Paritions数量](http://shanling2004.com/post/kafka-topicsparitions.html)
11. [随机和顺序访问 磁盘和内存的比较](http://queue.acm.org/detail.cfm?id=1563874)

