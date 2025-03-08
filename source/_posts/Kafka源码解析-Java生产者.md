---
title: Kafka源码解析-Java生产者
date: 2023-02-08 22:04:56
tags:
---

# Kafka生产流程

Quick Started

```java
        Properties props=new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG,KafkaProperties.KAFKA_SERVER_URL+":"+KafkaProperties.KAFKA_SERVER_PORT);
        props.put(ProducerConfig.CLIENT_ID_CONFIG,"DemoProducer");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,IntegerSerializer.class.getName());
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,StringSerializer.class.getName());
        props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG,true);
        producer=new KafkaProducer<>(props);

        producer.send(new ProducerRecord<>(topic,
        messageKey,
        messageStr),new Callback(){
@Override
public void onCompletion(RecordMetadata metadata,Exception exception){
        if(metadata!=null){
        System.out.println(
        "message("+key+", "+message+") sent to partition("+metadata.partition()+
        "), "+
        "offset("+metadata.offset()+") in "+elapsedTime+" ms");
        }else{
        exception.printStackTrace();
        }
        }
        });

        producer.close();
```

# 配置

```yaml
# 发送block时间. 在调用send时，block在请求partitionsFor()获取metadata和sendbuffer满的时候. 
max.block.ms=30000
  # The total bytes of memory the producer can use to buffer records waiting to be sent to the server. 
  # If records are sent faster than they can be delivered to the server the producer will block for **max.block.ms** after which it will throw an exception.
  # This setting should correspond roughly to the total memory the producer will use, but is not a hard bound since not all memory the producer uses is used for buffering. Some additional memory will be used for compression (if compression is enabled) as well as for maintaining in-flight requests.
buffer.memory=33554432 # 32M

```

# 源码解析
核心属性. 
###
``` java
public class KafkaProducer<K, V> implements Producer<K, V> {

    private final String clientId; // clientId，用于标示
    // Visible for testing
    final Metrics metrics; // metrics用于指标记录
    private final Partitioner partitioner; // 分区器，用于给message设置合理的分区
    private final int maxRequestSize; // 最大消息大小. 指单条最大的大小
    private final long totalMemorySize; 最大RecordAccumulator可以使用的内存buffer
    private final ProducerMetadata metadata; // 元数据，用于表示 topic和broker的关系
    private final RecordAccumulator accumulator;
    private final Sender sender;
    private final Thread ioThread;
    private final CompressionType compressionType;
    private final Sensor errors;
    private final Time time;
    private final Serializer<K> keySerializer;
    private final Serializer<V> valueSerializer;
    private final ProducerConfig producerConfig;
    private final long maxBlockTimeMs;
    private final ProducerInterceptors<K, V> interceptors;
    private final ApiVersions apiVersions;
    private final TransactionManager transactionManager;
}
```

### 发送主流程

```java

public Future<RecordMetadata> send(ProducerRecord<K, V> record,Callback callback){
        // **1. intercept the record**, which can be potentially modified; this method does not throw exceptions
        ProducerRecord<K, V> interceptedRecord=this.interceptors.onSend(record);
        return doSend(interceptedRecord,callback);
        }
```

发送选择分区

```java
public interface Partitioner extends Configurable, Closeable {
    int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster);
}
```

- `DefaultPartitioner`
    - If a partition is specified in the record, use it
      If no partition is specified but a key is present choose a partition based on a hash of the key
      If no partition or key is present choose the sticky partition that changes when the batch is full. See KIP-480 for
      details about sticky partitioning.
- `RoundRobinPartitioner`
- `UniformStickyPartitioner`

```java
private Future<RecordMetadata> doSend(ProducerRecord<K, V> record,Callback callback){
        TopicPartition tp=null;
        try{
        throwIfProducerClosed();
        // 1. **first make sure the metadata for the topic is available**
        long nowMs=time.milliseconds();
        ClusterAndWaitTime clusterAndWaitTime;
        try{
        clusterAndWaitTime=waitOnMetadata(record.topic(),record.partition(),nowMs,maxBlockTimeMs);
        }catch(KafkaException e){
        if(metadata.isClosed())
        throw new KafkaException("Producer closed while send in progress",e);
        throw e;
        }
        nowMs+=clusterAndWaitTime.waitedOnMetadataMs;
        ...
        // **2. 序列化**
        ...

        // **3. 选择分区**
        int partition=partition(record,serializedKey,serializedValue,cluster);
        tp=new TopicPartition(record.topic(),partition);

        // 验证消息大小. 
        int serializedSize=AbstractRecords.estimateSizeInBytesUpperBound(apiVersions.maxUsableProduceMagic(),
        compressionType,serializedKey,serializedValue,headers);
        ensureValidRecordSize(serializedSize);

        long timestamp=record.timestamp()==null?nowMs:record.timestamp();
        // producer callback will make sure to call both 'callback' and interceptor callback
        Callback interceptCallback=new InterceptorCallback<>(callback,this.interceptors,tp);

        if(transactionManager!=null&&transactionManager.isTransactional()){
        transactionManager.failIfNotReadyForSend();
        }
        // **4. Attempting to append record**
        RecordAccumulator.RecordAppendResult result=accumulator.append(tp,timestamp,serializedKey,
        serializedValue,headers,interceptCallback,remainingWaitMs,true,nowMs);

        if(result.abortForNewBatch){
        int prevPartition=partition;
        partitioner.onNewBatch(record.topic(),cluster,prevPartition);
        partition=partition(record,serializedKey,serializedValue,cluster);
        tp=new TopicPartition(record.topic(),partition);
        interceptCallback=new InterceptorCallback<>(callback,this.interceptors,tp);

        result=accumulator.append(tp,timestamp,serializedKey,
        serializedValue,headers,interceptCallback,remainingWaitMs,false,nowMs);
        }

        if(transactionManager!=null&&transactionManager.isTransactional())
        transactionManager.maybeAddPartitionToTransaction(tp);

        if(result.batchIsFull||result.newBatchCreated){
        log.trace("**Waking up the sender** since topic {} partition {} is either **full** or **getting a new batch**",record.topic(),partition);
        this.sender.wakeup();
        }
        return result.future;
        // **handling exceptions and record the errors**;
        // for API exceptions return them in the future,
        // for other exceptions throw directly
        }catch(...){
        }
        }

```

1. 首先根据订阅的topic来获取Metadata
2. 序列化
3. 选择要发送分区
4. AppendRecord，将消息Append到对应分区的Batch里面. `**ConcurrentMap<TopicPartition, Deque<ProducerBatch>> batches;**`