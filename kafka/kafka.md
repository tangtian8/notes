Kafka是一个分布式系统，由通过高性能[TCP网络协议进行](https://kafka.apache.org/protocol.html)通信的**服务器**和**客户端**组成。

**服务器**：Kafka作为一台或多台服务器的集群运行，可以跨越多个数据中心或云区域。其中一些服务器构成了存储层，称为代理。

**客户端**：它们使您可以编写分布式应用程序和微服务，即使在网络问题或机器故障的情况下，它们也可以并行，大规模且以容错的方式读取，写入和处理事件流。

在Kafka中，生产者和消费者之间完全脱钩且彼此不可知，这是实现Kafka众所周知的高可伸缩性的关键设计元素。例如，生产者永远不需要等待消费者。Kafka提供各种[保证，](https://kafka.apache.org/documentation/#intro_guarantees)例如能够一次准确地处理事件。

```bash
KStream<String, String> textLines = builder.stream("quickstart-events");

KTable<String, Long> wordCounts = textLines
            .flatMapValues(line -> Arrays.asList(line.toLowerCase().split(" ")))
            .groupBy((keyIgnored, word) -> word)
            .count();

wordCounts.toStream().to("output-topic"), Produced.with(Serdes.String(), Serdes.Long()));
```