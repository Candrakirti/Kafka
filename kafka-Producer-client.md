### 简介
Kafka可以看作是一个分布式高可用的消息队列。其Client端主要用Java语言实现，Server端则主要使用Scala来实现。这次，主要介绍Producer端的发送模型的实现。

### Producer-client的使用

在Producer-client中，我们主要设置是两个方面：设置Producer的属性来获得`KafkaProducer`实例，以及调用`send()`方法来将数据写入到消息队列中。

```java
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.clients.producer.Producer;
import java.util.Properties;

public class ProducerTest {
    private static String topicName;
    private static int msgNum;
    private static int key;

    public static void main(String[] args) {
        Properties props = new Properties();
        props.put("bootstrap.servers", "master:6667");
        props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");

        topicName = "test";
        msgNum = 10; // 发送的消息数

        Producer<String, String> producer = new KafkaProducer<>(props);
        for (int i = 0; i < msgNum; i++) {
            String msg = i + " Hello Kafka";
            producer.send(new ProducerRecord<String, String>(topicName, msg));
        }
        producer.close();
    }
}
```

### Producer数据发送流程

#### Producer的send实现

从以上的实例中，我们可以发现：用户给Kafka发送数据主要是调用Producer接口的`send()`方法。

```java
    Future<RecordMetadata> send(ProducerRecord<K, V> record);
    Future<RecordMetadata> send(ProducerRecord<K, V> record, Callback callback);
```

我们主要研究一下，KafkaProducer对于这个方法的实现：

```java
    //异步向一个topic发送数据
		@Override
    public Future<RecordMetadata> send(ProducerRecord<K, V> record) {
        return send(record, null);
    }
		// 向topic异步发送数据，当发送确认后唤起callback函数
		@Override
    public Future<RecordMetadata> send(ProducerRecord<K, V> record, Callback callback) {
        // intercept the record, which can be potentially modified; this method does not throw exceptions
        ProducerRecord<K, V> interceptedRecord = this.interceptors.onSend(record);
        return doSend(interceptedRecord, callback);
    }
```

很明显，`send()`方法执行的就是`doSend()`

#### doSend()方法的实现

```java
    /**
     * Implementation of asynchronously send a record to a topic.
     */
    private Future<RecordMetadata> doSend(ProducerRecord<K, V> record, Callback callback) {
        TopicPartition tp = null;
        try {
            throwIfProducerClosed();
            // 1.确认要发送到的topic的元数据是可用的
            ClusterAndWaitTime clusterAndWaitTime;
            try {
                clusterAndWaitTime = waitOnMetadata(record.topic(), record.partition(), maxBlockTimeMs);
            } catch (KafkaException e) {
                if (metadata.isClosed())
                    throw new KafkaException("Producer closed while send in progress", e);
                throw e;
            }
            long remainingWaitMs = Math.max(0, maxBlockTimeMs - clusterAndWaitTime.waitedOnMetadataMs);
            Cluster cluster = clusterAndWaitTime.cluster;
          // 2.序列化record的key和value
            byte[] serializedKey;
            try {
                serializedKey = keySerializer.serialize(record.topic(), record.headers(), record.key());
            } catch (ClassCastException cce) {
                throw new SerializationException("Can't convert key of class " + record.key().getClass().getName() +
                        " to class " + producerConfig.getClass(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG).getName() +
                        " specified in key.serializer", cce);
            }
            byte[] serializedValue;
            try {
                serializedValue = valueSerializer.serialize(record.topic(), record.headers(), record.value());
            } catch (ClassCastException cce) {
                throw new SerializationException("Can't convert value of class " + record.value().getClass().getName() +
                        " to class " + producerConfig.getClass(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG).getName() +
                        " specified in value.serializer", cce);
            }
          
          //3. 获取该record的partition值
            int partition = partition(record, serializedKey, serializedValue, cluster);
            tp = new TopicPartition(record.topic(), partition);
            setReadOnly(record.headers());
            Header[] headers = record.headers().toArray();

            int serializedSize = AbstractRecords.estimateSizeInBytesUpperBound(apiVersions.maxUsableProduceMagic(),
                    compressionType, serializedKey, serializedValue, headers);
            //如果record的字节超出限制或大于内存显示时，会抛出RecordTooLargeException
          	ensureValidRecordSize(serializedSize);
            long timestamp = record.timestamp() == null ? time.milliseconds() : record.timestamp();
            log.trace("Sending record {} with callback {} to topic {} partition {}", record, callback, record.topic(), partition);
            // producer callback will make sure to call both 'callback' and interceptor callback
            Callback interceptCallback = new InterceptorCallback<>(callback, this.interceptors, tp);

            if (transactionManager != null && transactionManager.isTransactional())
                transactionManager.maybeAddPartitionToTransaction(tp);
						//4.向Accumulator中追加数据
            RecordAccumulator.RecordAppendResult result = accumulator.append(tp, timestamp, serializedKey,
                    serializedValue, headers, interceptCallback, remainingWaitMs);
            //5.如果batch已经满了，唤醒sender线程发送数据
          	if (result.batchIsFull || result.newBatchCreated) {
                log.trace("Waking up the sender since topic {} partition {} is either full or getting a new batch", record.topic(), partition);
                //唤醒sender线程
              	this.sender.wakeup();
            }
            return result.future;
            // handling exceptions and record the errors;
            // for API exceptions return them in the future,
            // for other exceptions throw directly
        } catch (ApiException e) {
            log.debug("Exception occurred during message send:", e);
            if (callback != null)
                callback.onCompletion(null, e);
            this.errors.record();
            this.interceptors.onSendError(record, tp, e);
            return new FutureFailure(e);
        } catch (InterruptedException e) {
            this.errors.record();
            this.interceptors.onSendError(record, tp, e);
            throw new InterruptException(e);
        } catch (BufferExhaustedException e) {
            this.errors.record();
            this.metrics.sensor("buffer-exhausted-records").record();
            this.interceptors.onSendError(record, tp, e);
            throw e;
        } catch (KafkaException e) {
            this.errors.record();
            this.interceptors.onSendError(record, tp, e);
            throw e;
        } catch (Exception e) {
            // we notify interceptor about all exceptions, since onSend is called before anything else in this method
            this.interceptors.onSendError(record, tp, e);
            throw e;
        }
    }
```

根据以上代码分析，在`doSend()`方法中，一条Record数据的发送，可以分为以下几个步骤：

1. 确认数据发送到topic的metadata是可用的(如果该partition的leader存在则是可用的)，如果没有topic的metadata信息，就需要获取相应的metadata.
2. 序列化record的key和value
3. 获取record发送到的partition
4. 向accumulator中追加record数据，这个数据会先缓存到buffer
5. 追加完数据后，如果达到batch.size的大小，则唤醒`sender`线程发送数据。

### 发送过程详解

#### 获取topic的metadata信息

Producer通过`waitOnMetadata()`方法来获取对应topic的metadata信息。

首先，我们可以看下metadata(`org.apache.kafka.clients.Metadata`)的信息内容：

```java
    private static final Logger log = LoggerFactory.getLogger(Metadata.class);

    public static final long TOPIC_EXPIRY_MS = 5 * 60 * 1000;
    private static final long TOPIC_EXPIRY_NEEDS_UPDATE = -1L;

    private final long refreshBackoffMs;//更新最小间隔,默认100ms
    private final long metadataExpireMs;//metadata过期时间,默认60,000ms
    private int version;//每更新一次version自增一
    private long lastRefreshMs;//最近一次更新的时间
    private long lastSuccessfulRefreshMs;//最近一次成功更新时间
    private AuthenticationException authenticationException;
    private Cluster cluster;//集群中topic的信息
    private boolean needUpdate;
    /* Topics with expiry time */
    private final Map<String, Long> topics;//topic与其过期时间的对应关系
    private final List<Listener> listeners;//事件监控者
    private final ClusterResourceListeners clusterResourceListeners;//当接收到metadata更新时，ClusterResourceListeners的列表
    private boolean needMetadataForAllTopics;//是否强制更新所有的metadata
    private final boolean allowAutoTopicCreation;//是否允许自动创建topic
    private final boolean topicExpiryEnabled;//默认为ture, Producer会定期删除过期的topic
    private boolean isClosed;
```

关于topic的详细信息则都是在`Cluster`实例中保存的

```java
//org.apache.kafka.common.Cluster
    private final boolean isBootstrapConfigured;
    private final List<Node> nodes;
    private final Set<String> unauthorizedTopics;
    private final Set<String> internalTopics;
    private final Node controller;
    private final Map<TopicPartition, PartitionInfo> partitionsByTopicPartition;
    private final Map<String, List<PartitionInfo>> partitionsByTopic;
    private final Map<String, List<PartitionInfo>> availablePartitionsByTopic;
    private final Map<Integer, List<PartitionInfo>> partitionsByNode;
    private final Map<Integer, Node> nodesById;
    private final ClusterResource clusterResource;

//org.apache.kafka.common.PartitionInfo
    private final String topic;
    private final int partition;
    private final Node leader;
    private final Node[] replicas;
    private final Node[] inSyncReplicas;
    private final Node[] offlineReplicas;
```

根据以上信息，我们可以得出`cluster`实例主要是保存：

1. broker.id与node的对应关系
2. topic与partition(`PartitionInfo`)的对应关系
3. `node`与partition(`PartitionInfo`)的对应关系

##### Producer对于Metadata更新流程

```java
    private ClusterAndWaitTime waitOnMetadata(String topic, Integer partition, long maxWaitMs) throws InterruptedException {
        //在metadata中添加topic后，如果metadata中没有这个topic的meta,那么metadata的更新标志设置为true
        metadata.add(topic);
        Cluster cluster = metadata.fetch();
        Integer partitionsCount = cluster.partitionCountForTopic(topic);
        //当前metadata中如果已经有了这个topic的meta，就直接返回
        if (partitionsCount != null && (partition == null || partition < partitionsCount))
            return new ClusterAndWaitTime(cluster, 0);

        long begin = time.milliseconds();
        long remainingWaitMs = maxWaitMs;
        long elapsed;
				// 发送metadata请求，直到获取了这个topic的metadata或者请求超时
        do {
            log.trace("Requesting metadata update for topic {}.", topic);
            metadata.add(topic);
            int version = metadata.requestUpdate();
            sender.wakeup();
            try {
                metadata.awaitUpdate(version, remainingWaitMs);
            } catch (TimeoutException ex) {
                // Rethrow with original maxWaitMs to prevent logging exception with remainingWaitMs
                throw new TimeoutException("Failed to update metadata after " + maxWaitMs + " ms.");
            }
            cluster = metadata.fetch();
            elapsed = time.milliseconds() - begin;
            if (elapsed >= maxWaitMs)
                throw new TimeoutException("Failed to update metadata after " + maxWaitMs + " ms.");
            //认证失败，对当前topic没有Write的权限
          	if (cluster.unauthorizedTopics().contains(topic))
                throw new TopicAuthorizationException(topic);
            remainingWaitMs = maxWaitMs - elapsed;
            partitionsCount = cluster.partitionCountForTopic(topic);
        } while (partitionsCount == null);

        if (partition != null && partition >= partitionsCount) {
            throw new KafkaException(
                    String.format("Invalid partition given with record: %d is not in the range [0...%d).", partition, partitionsCount));
        }

        return new ClusterAndWaitTime(cluster, elapsed);
    }
```

在发送`metadata`请求的时候，主要做了以下操作：

1. `metadata.requestUpdate()`将metadata的needUpdate变量设置为true,并返回当前的版本号(`Version`), 通过版本号来判断metadata是否完成更新
2. `sender.wakeup()`唤醒`sender`线程, `sender`线程又去唤醒`NetworkClient`线程, `NetworkClient`线程进行一些实际的操作
3. `metadata.awaitUpdate(version,remianingWaitMs)`等待metadata的更新

#### Key和Value的序列化

Producer-client对record的`key`和`value`值进行序列化操作，而在`comsumer-client`中则进行反序列化。

序列化和反序列化代码可以看`org.apache.kafka.common.serialization`下面的文件

#### 获取partition值

关于partition值的计算，主要分为以下三种情况：

1. 用户指明了要使用的`parition`时，直接将用户指明的值作为parition
2. 没指明`partition`值，有`key`值: 将`hash(key)%partition_num`的值作为parition
3. 没指明`parition`、`key`值：第一次调用时随机生成一个整数（后面每次调用则在这个整数上自增)num, 将`num%parition_num`作为parition值（`round-robin算法`）

在`producer-client`中，具体实现如下：

```java
    private int partition(ProducerRecord<K, V> record, byte[] serializedKey, byte[] serializedValue, Cluster cluster) {
        Integer partition = record.partition();
        return partition != null ?
                partition :
                partitioner.partition(
                        record.topic(), record.key(), serializedKey, record.value(), serializedValue, cluster);
      //若没指定parition,运行partitioner.partition
    }
```

默认使用的方法`org.apache.kafka.clients.producer.internals.DefaultPartitioner`的`partition`

```java
    public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
        List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
        int numPartitions = partitions.size();
        if (keyBytes == null) {
            int nextValue = nextValue(topic);
            List<PartitionInfo> availablePartitions = cluster.availablePartitionsForTopic(topic);
            if (availablePartitions.size() > 0) {
                int part = Utils.toPositive(nextValue) % availablePartitions.size();
                return availablePartitions.get(part).partition();
            } else {
                // no partitions are available, give a non-available partition
                return Utils.toPositive(nextValue) % numPartitions;
            }
        } else {··
            // 将`hash(key)%partition_num`的值作为parition
            return Utils.toPositive(Utils.murmur2(keyBytes)) % numPartitions;
        }
    }
```

向Accumulator写数据

根据上面代码的分析，我们可以知道`producer-client`向Server写数据，主要是：获取topic的metadat,然后将ke y和value序列化，然后获取partition值，最后将数据写入到buffer中。当buffer达到某个阈值时，系统就会唤起`sender()`线程区发送`RecordBatch`。在下面将分析Producer是如何向Buffer中写入数据的。

在RecordAccumulator中，有一个十分重要的变量`ConcurrentMap<TopicPartition, Deque<RecordBatch>> batches`. 这个变量维护的主要是每个`topic-partition`的队列。换而言之，当添加数据的时候，kafka-client会向其对应的topic-partition对应的queue最新创建的一个RecordBatch中添加record。当然，由于队列的FiFO特性，在发送的时候，他会将最老的数据发送出去。

```java
    public RecordAppendResult append(TopicPartition tp,
                                     long timestamp,
                                     byte[] key,
                                     byte[] value,
                                     Header[] headers,
                                     Callback callback,
                                     long maxTimeToBlock) throws InterruptedException {
        // We keep track of the number of appending thread to make sure we do not miss batches in
        // abortIncompleteBatches().
        appendsInProgress.incrementAndGet();
        ByteBuffer buffer = null;
        if (headers == null) headers = Record.EMPTY_HEADERS;
        try {
            // check if we have an in-progress batch
            Deque<ProducerBatch> dq = getOrCreateDeque(tp);
            synchronized (dq) {
                if (closed)
                    throw new KafkaException("Producer closed while send in progress");
                RecordAppendResult appendResult = tryAppend(timestamp, key, value, headers, callback, dq);
                if (appendResult != null)
                    return appendResult;
            }

            // we don't have an in-progress record batch try to allocate a new batch
            byte maxUsableMagic = apiVersions.maxUsableProduceMagic();
          //初始化buffer,为其分配的大小max（bach.size,加上头文件的本条消息的大小的压缩后大小）内存
            int size = Math.max(this.batchSize, AbstractRecords.estimateSizeInBytesUpperBound(maxUsableMagic, compression, key, value, headers));
            log.trace("Allocating a new {} byte message buffer for topic {} partition {}", size, tp.topic(), tp.partition());
            buffer = free.allocate(size, maxTimeToBlock);
            synchronized (dq) {
                // Need to check if producer is closed again after grabbing the dequeue lock.
                if (closed)
                    throw new KafkaException("Producer closed while send in progress");

                RecordAppendResult appendResult = tryAppend(timestamp, key, value, headers, callback, dq);
                if (appendResult != null) {
                    // Somebody else found us a batch, return the one we waited for! Hopefully this doesn't happen often...
									//如果发现这个queue存在，那就返回这个queue
                  return appendResult;
                }
								// 给topic-partition创建一个新的RecordBatch
                MemoryRecordsBuilder recordsBuilder = recordsBuilder(buffer, maxUsableMagic);
                ProducerBatch batch = new ProducerBatch(tp, recordsBuilder, time.milliseconds());
              // 向新的RecordBatch追加数据
                FutureRecordMetadata future = Utils.notNull(batch.tryAppend(timestamp, key, value, headers, callback, time.milliseconds()));

                dq.addLast(batch);//加入到对应的queue中
                incomplete.add(batch);// 向未ack的batch集合添加这个batch

                // Don't deallocate this buffer in the finally block as it's being used in the record batch
                buffer = null;
                return new RecordAppendResult(future, dq.size() > 1 || batch.isFull(), true);
            }
        } finally {
            if (buffer != null)
                free.deallocate(buffer);
            appendsInProgress.decrementAndGet();
        }
    }
```

根据以上代码分析，大概可以总结到record写入的步骤大致如下：

1. 获取该topic-partition对应的queue,没有的话创建一个空的queue
2. 向queue中追加数据
3. 创建buffer
4. 向新的RecordBatch中添加数据
5. 向新建的RecordBatch写入record,并将RecordBatch添加到queue中，返回结果，写入成功

### 发送RecordBatch

当record写入成功后，如果发现RecordBatch满足发送条件，则会唤醒sender线程，发送RecordBatch。

sender线程发送数据的实现如下：

```java
    private long sendProducerData(long now) {
        Cluster cluster = metadata.fetch();

        // get the list of partitions with data ready to send
        RecordAccumulator.ReadyCheckResult result = this.accumulator.ready(cluster, now);

        // if there are any partitions whose leaders are not known yet, force metadata update
        if (!result.unknownLeaderTopics.isEmpty()) {
            // The set of topics with unknown leader contains topics with leader election pending as well as
            // topics which may have expired. Add the topic again to metadata to ensure it is included
            // and request metadata update, since there are messages to send to the topic.
            for (String topic : result.unknownLeaderTopics)
                this.metadata.add(topic);

            log.debug("Requesting metadata update due to unknown leader topics from the batched records: {}", result.unknownLeaderTopics);

            this.metadata.requestUpdate();
        }

        // remove any nodes we aren't ready to send to
        Iterator<Node> iter = result.readyNodes.iterator();
        long notReadyTimeout = Long.MAX_VALUE;
        while (iter.hasNext()) {
            Node node = iter.next();
            if (!this.client.ready(node, now)) {
                iter.remove();
                notReadyTimeout = Math.min(notReadyTimeout, this.client.pollDelayMs(node, now));
            }
        }

        // create produce requests
        Map<Integer, List<ProducerBatch>> batches = this.accumulator.drain(cluster, result.readyNodes,
                this.maxRequestSize, now);
        if (guaranteeMessageOrder) {
            // Mute all the partitions drained
            for (List<ProducerBatch> batchList : batches.values()) {
                for (ProducerBatch batch : batchList)
                    this.accumulator.mutePartition(batch.topicPartition);
            }
        }

        List<ProducerBatch> expiredBatches = this.accumulator.expiredBatches(this.requestTimeoutMs, now);
        // Reset the producer id if an expired batch has previously been sent to the broker. Also update the metrics
        // for expired batches. see the documentation of @TransactionState.resetProducerId to understand why
        // we need to reset the producer id here.
        if (!expiredBatches.isEmpty())
            log.trace("Expired {} batches in accumulator", expiredBatches.size());
        for (ProducerBatch expiredBatch : expiredBatches) {
            failBatch(expiredBatch, -1, NO_TIMESTAMP, expiredBatch.timeoutException(), false);
            if (transactionManager != null && expiredBatch.inRetry()) {
                // This ensures that no new batches are drained until the current in flight batches are fully resolved.
                transactionManager.markSequenceUnresolved(expiredBatch.topicPartition);
            }
        }

        sensors.updateProduceRequestMetrics(batches);

        // If we have any nodes that are ready to send + have sendable data, poll with 0 timeout so this can immediately
        // loop and try sending more data. Otherwise, the timeout is determined by nodes that have partitions with data
        // that isn't yet sendable (e.g. lingering, backing off). Note that this specifically does not include nodes
        // with sendable data that aren't ready to send since they would cause busy looping.
        long pollTimeout = Math.min(result.nextReadyCheckDelayMs, notReadyTimeout);
        if (!result.readyNodes.isEmpty()) {
            log.trace("Nodes with data ready to send: {}", result.readyNodes);
            // if some partitions are already ready to be sent, the select time would be 0;
            // otherwise if some partition already has some data accumulated but not ready yet,
            // the select time will be the time difference between now and its linger expiry time;
            // otherwise the select time will be the time difference between now and the metadata expiry time;
            pollTimeout = 0;
        }
        sendProduceRequests(batches, now);

        return pollTimeout;
    }
```



