Kafka中的每个partition都由一系列有序的、不可变的消息组成，这些消息被连续的追加到partition中。partition中的每个消息都有一个连续的序号，用于partition唯一标识一条消息。

**Offset记录着下一条将要发送给Consumer的消息的序号。**

Offset从语义上来看拥有两种：Current Offset和Committed Offset。

# Current Offset

Current Offset保存在Consumer客户端中，它表示Consumer希望收到的下一条消息的序号。它仅仅在poll()方法中使用。例如，Consumer第一次调用poll()方法后收到了20条消息，那么Current Offset就被设置为20。这样Consumer下一次调用poll()方法时，Kafka就知道应该从序号为21的消息开始读取。这样就能够保证每次Consumer poll消息时，都能够收到不重复的消息。

# Committed Offset

Committed Offset保存在Broker上，它表示Consumer已经确认消费过的消息的序号。主要通过[`commitSync`](https://links.jianshu.com/go?to=https%3A%2F%2Fkafka.apache.org%2F10%2Fjavadoc%2Forg%2Fapache%2Fkafka%2Fclients%2Fconsumer%2FKafkaConsumer.html%23commitSync--)和[`commitAsync`](https://links.jianshu.com/go?to=https%3A%2F%2Fkafka.apache.org%2F10%2Fjavadoc%2Forg%2Fapache%2Fkafka%2Fclients%2Fconsumer%2FKafkaConsumer.html%23commitAsync-org.apache.kafka.clients.consumer.OffsetCommitCallback-)
 API来操作。举个例子，Consumer通过poll() 方法收到20条消息后，此时Current Offset就是20，经过一系列的逻辑处理后，并没有调用`consumer.commitAsync()`或`consumer.commitSync()`来提交Committed Offset，那么此时Committed Offset依旧是0。

Committed Offset主要用于Consumer Rebalance。在Consumer Rebalance的过程中，一个partition被分配给了一个Consumer，那么这个Consumer该从什么位置开始消费消息呢？答案就是Committed Offset。另外，如果一个Consumer消费了5条消息（poll并且成功commitSync）之后宕机了，重新启动之后它仍然能够从第6条消息开始消费，因为Committed Offset已经被Kafka记录为5。

**总结一下，Current Offset是针对Consumer的poll过程的，它可以保证每次poll都返回不重复的消息；而Committed Offset是用于Consumer Rebalance过程的，它能够保证新的Consumer能够从正确的位置开始消费一个partition，从而避免重复消费。**

在Kafka 0.9前，Committed Offset信息保存在zookeeper的[consumers/{group}/offsets/{topic}/{partition}]目录中（zookeeper其实并不适合进行大批量的读写操作，尤其是写操作）。而在0.9之后，所有的offset信息都保存在了Broker上的一个名为__consumer_offsets的topic中。

Kafka集群中offset的管理都是由Group Coordinator中的Offset Manager完成的。

# Group Coordinator

Group Coordinator是运行在Kafka集群中每一个Broker内的一个进程。它主要负责Consumer Group的管理，Offset位移管理以及[Consumer Rebalance](https://www.jianshu.com/p/d42b7a4ac8ad)。

对于每一个Consumer Group，Group Coordinator都会存储以下信息：

- 订阅的topics列表
- Consumer Group配置信息，包括session timeout等
- 组中每个Consumer的元数据。包括主机名，consumer id
- 每个Group正在消费的topic partition的当前offsets
- Partition的ownership元数据，包括consumer消费的partitions映射关系

Consumer Group如何确定自己的coordinator是谁呢？ 简单来说分为两步：

1. 确定Consumer Group offset信息将要写入__consumers_offsets topic的哪个分区。具体计算公式：



```swift
__consumers_offsets partition# = Math.abs(groupId.hashCode() % offsets.topic.num.partitions)  //offsets.topic.num.partitions默认值为50。
```

1. 该分区leader所在的broker就是被选定的coordinator

# Offset存储模型

由于一个partition只能固定的交给一个消费者组中的一个消费者消费，因此Kafka保存offset时并不直接为每个消费者保存，而是以groupid-topic-partition -> offset的方式保存。如图所示：



![img](https:////upload-images.jianshu.io/upload_images/448235-d4168722d7c9e0e9.png?imageMogr2/auto-orient/strip|imageView2/2/w/850/format/webp)

group-offset.png

**Kafka在保存Offset的时候，实际上是将Consumer Group和partition对应的offset以消息的方式保存在__consumers_offsets这个topic中**。

__consumers_offsets默认拥有50个partition，可以通过



```css
Math.abs(groupId.hashCode() % offsets.topic.num.partitions) 
```

的方式来查询某个Consumer Group的offset信息保存在__consumers_offsets的哪个partition中。下图展示了__consumers_offsets中保存的offset消息的格式：



![img](https:////upload-images.jianshu.io/upload_images/448235-f035acea5b391d77.png?imageMogr2/auto-orient/strip|imageView2/2/w/663/format/webp)

__consumers_offsets.png

![img](https:////upload-images.jianshu.io/upload_images/448235-1bf415dfc0391702.png?imageMogr2/auto-orient/strip|imageView2/2/w/614/format/webp)

__consumers_offsets_data.png

如图所示，一条offset消息的格式为groupid-topic-partition -> offset。因此consumer poll消息时，已知groupid和topic，又通过Coordinator分配partition的方式获得了对应的partition，自然能够通过Coordinator查找__consumers_offsets的方式获得最新的offset了。

# Offset查询

前面我们已经描述过offset的存储模型，它是按照**groupid-topic-partition -> offset**的方式存储的。然而Kafka只提供了根据offset读取消息的模型，并不支持根据key读取消息的方式。那么Kafka是如何支持Offset的查询呢？

**答案就是Offsets Cache！！**

![img](https:////upload-images.jianshu.io/upload_images/448235-0a26fa0cb627dd86.JPG?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

Offsets Cache.JPG



如图所示，Consumer提交offset时，Kafka Offset Manager会首先追加一条条新的commit消息到__consumers_offsets topic中，然后更新对应的缓存。读取offset时从缓存中读取，而不是直接读取__consumers_offsets这个topic。

# Log Compaction

我们已经知道，Kafka使用*groupid-topic-partition -> offset**的消息格式，将Offset信息存储在__consumers_offsets topic中。请看下面一个例子：

![img](https:////upload-images.jianshu.io/upload_images/448235-f7885881b97f96ae.JPG?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

__consumers_offsets.JPG

如图，对于audit-consumer这个Consumer Group来说，上面的存储了两条具有相同key的记录：`PageViewEvent-0 -> 240`和`PageViewEvent-0 -> 323`。事实上，这就是一种无用的冗余。因为对于一个partition来说，我们实际上只需要它当前最新的Offsets。因此这条旧的`PageViewEvent-0 -> 240`记录事实上是无用的。

为了消除这样的过期数据，Kafka为__consumers_offsets topic设置了Log Compaction功能。Log Compaction意味着对于有相同key的的不同value值，只保留最后一个版本。如果应用只关心key对应的最新value值，可以开启Kafka的Log Compaction功能，Kafka会定期将相同key的消息进行合并，只保留最新的value值。

这张图片生动的阐述了Log Compaction的过程：



![img](https:////upload-images.jianshu.io/upload_images/448235-0f408def937f9a95.JPG?imageMogr2/auto-orient/strip|imageView2/2/w/1115/format/webp)

Log Compaction.JPG

下图阐释了__consumers_offsets topic中的数据在Log Compaction下的变化：



![img](https:////upload-images.jianshu.io/upload_images/448235-303f434e841451f5.JPG?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

Log Compaction for __consumers_offsets.JPG

> **在新建topic时添加`log.cleanup.policy=compact`参数就可以为topic开启Log Compaction功能。**

# auto.offset.reset参数

`auto.offset.reset`表示如果Kafka中没有存储对应的offset信息的话（有可能offset信息被删除），消费者从何处开始消费消息。它拥有三个可选值：

- earliest：从最早的offset开始消费
- latest：从最后的offset开始消费
- none：直接抛出exception给consumer

看一下下面两个场景：

1. Consumer消费了5条消息后宕机了，重启之后它读取到对应的partition的Committed Offset为5，因此会直接从第6条消息开始读取。此时完全依赖于Committed Offset机制，和`auto.offset.reset`配置完全无关。
2. 新建了一个新的Group，并添加了一个Consumer，它订阅了一个已经存在的Topic。此时Kafka中还没有这个Consumer相应的Offset信息，因此此时Kafka就会根据`auto.offset.reset`配置来决定这个Consumer从何处开始消费消息。

### 参考文章

- [https://www.cnblogs.com/wq3435/p/8001094.html](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.cnblogs.com%2Fwq3435%2Fp%2F8001094.html)



18人点赞



[Kafka从入门到实践]()





作者：伊凡的一天
链接：https://www.jianshu.com/p/449074d97daf
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。