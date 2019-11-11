# Kafka topic的学习

### topic的概念

`topic`是`kafka`中的一个消息队列的标识. 换而言之, `topic`用来区分不同的消息队列. 一个`topic`是由若干个`partition`构成的, 这些`partition`通常是分布在不同的多台`broker`上面的. 同时, 由于kafka是一个分布式高可用消息队列, 这就需要多个replica来保证`fault tolerance`.

### Topic的创建

对于`topic`的创建, Kafka主要提供了以下两种方式:

1. 通过`kafka-topics.sh`创建一个`topic` 可以设置相应的replica和partition

   `./bin/kafka-topics.sh --create --topic test --zookeeper xxx --partitions 3 --replication-facotr 2`

2. Server端如果`auto.create.topics.enable`设置为true时, 那么当`producer`向一个不存在的`topic`发送数据时, 该`topic`同样会被创建出来, 此时, 副本数默认是1

