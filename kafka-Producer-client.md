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
        } else {
            // 将`hash(key)%partition_num`的值作为parition
            return Utils.toPositive(Utils.murmur2(keyBytes)) % numPartitions;
        }
    }
```

