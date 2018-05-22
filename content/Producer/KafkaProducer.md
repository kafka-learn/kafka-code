---

title: "KafkaProducer"
date: 2018-5-22 19:06

---

[TOC]

# KafkaProducer类的实现

```java
package org.apache.kafka.clients.producer;

import org.apache.kafka.clients.ApiVersions;
import org.apache.kafka.clients.ClientUtils;
import org.apache.kafka.clients.KafkaClient;
import org.apache.kafka.clients.Metadata;
import org.apache.kafka.clients.NetworkClient;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.clients.consumer.OffsetAndMetadata;
import org.apache.kafka.clients.consumer.OffsetCommitCallback;
import org.apache.kafka.clients.producer.internals.ProducerInterceptors;
import org.apache.kafka.clients.producer.internals.ProducerMetrics;
import org.apache.kafka.clients.producer.internals.RecordAccumulator;
import org.apache.kafka.clients.producer.internals.Sender;
import org.apache.kafka.clients.producer.internals.TransactionManager;
import org.apache.kafka.clients.producer.internals.TransactionalRequestResult;
import org.apache.kafka.common.Cluster;
import org.apache.kafka.common.KafkaException;
import org.apache.kafka.common.Metric;
import org.apache.kafka.common.MetricName;
import org.apache.kafka.common.PartitionInfo;
import org.apache.kafka.common.TopicPartition;
import org.apache.kafka.common.config.ConfigException;
import org.apache.kafka.common.errors.ApiException;
import org.apache.kafka.common.errors.AuthenticationException;
import org.apache.kafka.common.errors.AuthorizationException;
import org.apache.kafka.common.errors.InterruptException;
import org.apache.kafka.common.errors.ProducerFencedException;
import org.apache.kafka.common.errors.RecordTooLargeException;
import org.apache.kafka.common.errors.SerializationException;
import org.apache.kafka.common.errors.TimeoutException;
import org.apache.kafka.common.errors.TopicAuthorizationException;
import org.apache.kafka.common.header.Header;
import org.apache.kafka.common.header.Headers;
import org.apache.kafka.common.header.internals.RecordHeaders;
import org.apache.kafka.common.internals.ClusterResourceListeners;
import org.apache.kafka.common.metrics.JmxReporter;
import org.apache.kafka.common.metrics.MetricConfig;
import org.apache.kafka.common.metrics.Metrics;
import org.apache.kafka.common.metrics.MetricsReporter;
import org.apache.kafka.common.metrics.Sensor;
import org.apache.kafka.common.network.ChannelBuilder;
import org.apache.kafka.common.network.Selector;
import org.apache.kafka.common.record.AbstractRecords;
import org.apache.kafka.common.record.CompressionType;
import org.apache.kafka.common.record.RecordBatch;
import org.apache.kafka.common.serialization.ExtendedSerializer;
import org.apache.kafka.common.serialization.Serializer;
import org.apache.kafka.common.utils.AppInfoParser;
import org.apache.kafka.common.utils.KafkaThread;
import org.apache.kafka.common.utils.LogContext;
import org.apache.kafka.common.utils.Time;
import org.slf4j.Logger;

import java.net.InetSocketAddress;
import java.util.Collections;
import java.util.List;
import java.util.Map;
import java.util.Objects;
import java.util.Properties;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.Future;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.atomic.AtomicReference;

import static org.apache.kafka.common.serialization.ExtendedSerializer.Wrapper.ensureExtended;

public class KafkaProducer<K, V> implements Producer<K, V> {

    //调用Slf4j的接口API
    private final Logger log;

    //通过juc的Atomic类来保证生成唯一的客户端id
    private static final AtomicInteger PRODUCER_CLIENT_ID_SEQUENCE = new AtomicInteger(1);

    //规定的JMX监控程序的前缀
    private static final String JMX_PREFIX = "kafka.producer";

    //规定的网络线程的前缀
    public static final String NETWORK_THREAD_PREFIX = "kafka-producer-network-thread";

    //用户指定客户端id
    private final String clientId;

    // Visible for testing
    //生产者维护的完整的内部统计信息
    final Metrics metrics;

    //分区选择器，根据一定的策略将消息推送到选中的分区
    private final Partitioner partitioner;

    //消息的最大长度（消息头+key+value）
    private final int maxRequestSize;

    //一条待发送的消息可占据的缓冲区大小
    private final long totalMemorySize;

    //kafka集群的元数据信息
    private final Metadata metadata;

    //消息暂存器
    private final RecordAccumulator accumulator;

    //负责发送向kafka集群发送生产出的消息的后台线程
    private final Sender sender;

    //执行sender线程发送任务的线程
    private final Thread ioThread;

    //消息压缩类型
    private final CompressionType compressionType;

    //
    private final Sensor errors;

    //表示时间
    private final Time time;

    //key的序列化器
    private final ExtendedSerializer<K> keySerializer;

    //value的序列化器
    private final ExtendedSerializer<V> valueSerializer;

    //存储生产者配置的对象
    private final ProducerConfig producerConfig;

    //等待更新kafka集群元数据信息的最大时长
    private final long maxBlockTimeMs;

    //发送请求等待ACK确认的最大时长
    private final int requestTimeoutMs;

    //生产者消息拦截器集合
    private final ProducerInterceptors<K, V> interceptors;

    //API版本
    private final ApiVersions apiVersions;

    //事务管理
    private final TransactionManager transactionManager;

    //Kafka生产者的五个构造函数，内部均调用第五个构造函数，没有的参数设置为null

    //接受键值对形式的配置参数
    public KafkaProducer(final Map<String, Object> configs) {
        this(new ProducerConfig(configs), null, null, null, null);
    }

    //接受键值对形式的配置参数，接受key和value的序列化器
    public KafkaProducer(Map<String, Object> configs, Serializer<K> keySerializer, Serializer<V> valueSerializer) {
        this(new ProducerConfig(ProducerConfig.addSerializerToConfig(configs, keySerializer, valueSerializer)),
            keySerializer,
            valueSerializer,
            null,
            null);
    }

    //以Properties对象的形式传入配置参数
    public KafkaProducer(Properties properties) {
        this(new ProducerConfig(properties), null, null, null, null);
    }

    //以Properties对象的形式传入配置参数，接受key和value的序列化器
    public KafkaProducer(Properties properties, Serializer<K> keySerializer, Serializer<V> valueSerializer) {
        this(new ProducerConfig(ProducerConfig.addSerializerToConfig(properties, keySerializer, valueSerializer)),
                keySerializer, valueSerializer, null, null);
    }

    //接受ProducerConfig类型的对象形式的参数，接受key和value的序列化器，接受元数据信息对象，接受Kafka客户端对象
    @SuppressWarnings("unchecked")
    // visible for testing
    KafkaProducer(ProducerConfig config,
                  Serializer<K> keySerializer,
                  Serializer<V> valueSerializer,
                  Metadata metadata,
                  KafkaClient kafkaClient) {
        try {
            //将用户传入的ProducerConfig类型的对象的数据以键值对的格式来存储
            Map<String, Object> userProvidedConfigs = config.originals();

            //存储ProducerConfig
            this.producerConfig = config;

            //存储当前时间
            this.time = Time.SYSTEM;

            //获取用户传入的客户端id
            String clientId = config.getString(ProducerConfig.CLIENT_ID_CONFIG);

            //如果用户没有传入，则用juc的Atomic自增对象来生成，并与固定前缀相连，赋给对象属性
            if (clientId.length() <= 0)
                clientId = "producer-" + PRODUCER_CLIENT_ID_SEQUENCE.getAndIncrement();
            this.clientId = clientId;

            //获取用户传入的事务id，如果用户没传，则为空值
            String transactionalId = userProvidedConfigs.containsKey(ProducerConfig.TRANSACTIONAL_ID_CONFIG) ?
                    (String) userProvidedConfigs.get(ProducerConfig.TRANSACTIONAL_ID_CONFIG) : null;

            //根据客户端id和事务id来生成相应的日志内容
            LogContext logContext;
            if (transactionalId == null)
                logContext = new LogContext(String.format("[Producer clientId=%s] ", clientId));
            else
                logContext = new LogContext(String.format("[Producer clientId=%s, transactionalId=%s] ", clientId, transactionalId));
            //为当前对象的log属性赋予上面生成的日志内容
            log = logContext.logger(KafkaProducer.class);

            //输出生产者开始工作的log信息
            log.trace("Starting the Kafka producer");

            //
            Map<String, String> metricTags = Collections.singletonMap("client-id", clientId);
            MetricConfig metricConfig = new MetricConfig().samples(config.getInt(ProducerConfig.METRICS_NUM_SAMPLES_CONFIG))
                    .timeWindow(config.getLong(ProducerConfig.METRICS_SAMPLE_WINDOW_MS_CONFIG), TimeUnit.MILLISECONDS)
                    .recordLevel(Sensor.RecordingLevel.forName(config.getString(ProducerConfig.METRICS_RECORDING_LEVEL_CONFIG)))
                    .tags(metricTags);

            //通过单例设置JMX监控
            List<MetricsReporter> reporters = config.getConfiguredInstances(ProducerConfig.METRIC_REPORTER_CLASSES_CONFIG,
                    MetricsReporter.class);
            reporters.add(new JmxReporter(JMX_PREFIX));

            //设置统计信息
            this.metrics = new Metrics(metricConfig, reporters, time);
            ProducerMetrics metricsRegistry = new ProducerMetrics(this.metrics);

            //通过单例设置分区选择器
            this.partitioner = config.getConfiguredInstance(ProducerConfig.PARTITIONER_CLASS_CONFIG, Partitioner.class);
            long retryBackoffMs = config.getLong(ProducerConfig.RETRY_BACKOFF_MS_CONFIG);

            //通过单例设置key和value的序列化器
            if (keySerializer == null) {
                this.keySerializer = ensureExtended(config.getConfiguredInstance(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,
                                                                                         Serializer.class));
                this.keySerializer.configure(config.originals(), true);
            } else {
                config.ignore(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG);
                this.keySerializer = ensureExtended(keySerializer);
            }
            if (valueSerializer == null) {
                this.valueSerializer = ensureExtended(config.getConfiguredInstance(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
                                                                                           Serializer.class));
                this.valueSerializer.configure(config.originals(), false);
            } else {
                config.ignore(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG);
                this.valueSerializer = ensureExtended(valueSerializer);
            }

            //设置客户端id
            // load interceptors and make sure they get clientId
            userProvidedConfigs.put(ProducerConfig.CLIENT_ID_CONFIG, clientId);

            //设置消息拦截器集合
            List<ProducerInterceptor<K, V>> interceptorList = (List) (new ProducerConfig(userProvidedConfigs, false)).getConfiguredInstances(ProducerConfig.INTERCEPTOR_CLASSES_CONFIG,
                    ProducerInterceptor.class);
            this.interceptors = new ProducerInterceptors<>(interceptorList);
            ClusterResourceListeners clusterResourceListeners = configureClusterResourceListeners(keySerializer, valueSerializer, interceptorList, reporters);

            //设置最大消息长度、发送单条消息的缓冲区大小和压缩类型
            this.maxRequestSize = config.getInt(ProducerConfig.MAX_REQUEST_SIZE_CONFIG);
            this.totalMemorySize = config.getLong(ProducerConfig.BUFFER_MEMORY_CONFIG);
            this.compressionType = CompressionType.forName(config.getString(ProducerConfig.COMPRESSION_TYPE_CONFIG));

            //设置等待更新元数据信息的最大时长、请求得到确认的最大时长和事务管理器
            this.maxBlockTimeMs = config.getLong(ProducerConfig.MAX_BLOCK_MS_CONFIG);
            this.requestTimeoutMs = config.getInt(ProducerConfig.REQUEST_TIMEOUT_MS_CONFIG);
            this.transactionManager = configureTransactionState(config, logContext, log);
            int retries = configureRetries(config, transactionManager != null, log);
            int maxInflightRequests = configureInflightRequests(config, transactionManager != null);
            short acks = configureAcks(config, transactionManager != null, log);

            this.apiVersions = new ApiVersions();

            //设置消息收集器
            this.accumulator = new RecordAccumulator(logContext,
                    config.getInt(ProducerConfig.BATCH_SIZE_CONFIG),
                    this.totalMemorySize,
                    this.compressionType,
                    config.getLong(ProducerConfig.LINGER_MS_CONFIG),
                    retryBackoffMs,
                    metrics,
                    time,
                    apiVersions,
                    transactionManager);

            //设置Kafka服务器的主机名和端口号
            List<InetSocketAddress> addresses = ClientUtils.parseAndValidateAddresses(config.getList(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG));
            if (metadata != null) {
                this.metadata = metadata;
            } else {
                this.metadata = new Metadata(retryBackoffMs, config.getLong(ProducerConfig.METADATA_MAX_AGE_CONFIG),
                    true, true, clusterResourceListeners);
                this.metadata.update(Cluster.bootstrap(addresses), Collections.<String>emptySet(), time.milliseconds());
            }
            //创建通道
            ChannelBuilder channelBuilder = ClientUtils.createChannelBuilder(config);
            Sensor throttleTimeSensor = Sender.throttleTimeSensor(metricsRegistry.senderMetrics);

            //创建Kafka的网络I/O客户端
            KafkaClient client = kafkaClient != null ? kafkaClient : new NetworkClient(
                    new Selector(config.getLong(ProducerConfig.CONNECTIONS_MAX_IDLE_MS_CONFIG),
                            this.metrics, time, "producer", channelBuilder, logContext),
                    this.metadata,
                    clientId,
                    maxInflightRequests,
                    config.getLong(ProducerConfig.RECONNECT_BACKOFF_MS_CONFIG),
                    config.getLong(ProducerConfig.RECONNECT_BACKOFF_MAX_MS_CONFIG),
                    config.getInt(ProducerConfig.SEND_BUFFER_CONFIG),
                    config.getInt(ProducerConfig.RECEIVE_BUFFER_CONFIG),
                    this.requestTimeoutMs,
                    time,
                    true,
                    apiVersions,
                    throttleTimeSensor,
                    logContext);

            //设置sender线程的相关参数
            this.sender = new Sender(logContext,
                    client,
                    this.metadata,
                    this.accumulator,
                    maxInflightRequests == 1,
                    config.getInt(ProducerConfig.MAX_REQUEST_SIZE_CONFIG),
                    acks,
                    retries,
                    metricsRegistry.senderMetrics,
                    Time.SYSTEM,
                    this.requestTimeoutMs,
                    config.getLong(ProducerConfig.RETRY_BACKOFF_MS_CONFIG),
                    this.transactionManager,
                    apiVersions);

            //设置sender的执行线程并启动
            String ioThreadName = NETWORK_THREAD_PREFIX + " | " + clientId;
            this.ioThread = new KafkaThread(ioThreadName, this.sender, true);
            this.ioThread.start();
            this.errors = this.metrics.sensor("errors");
            config.logUnused();
            AppInfoParser.registerAppInfo(JMX_PREFIX, clientId, metrics);
            log.debug("Kafka producer started");
        } catch (Throwable t) {
            // call close methods if internal objects are already constructed this is to prevent resource leak. see KAFKA-2121
            close(0, TimeUnit.MILLISECONDS, true);
            // now propagate the exception
            throw new KafkaException("Failed to construct kafka producer", t);
        }
    }

    private static TransactionManager configureTransactionState(ProducerConfig config, LogContext logContext, Logger log) {

        TransactionManager transactionManager = null;

        boolean userConfiguredIdempotence = false;
        if (config.originals().containsKey(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG))
            userConfiguredIdempotence = true;

        boolean userConfiguredTransactions = false;
        if (config.originals().containsKey(ProducerConfig.TRANSACTIONAL_ID_CONFIG))
            userConfiguredTransactions = true;

        boolean idempotenceEnabled = config.getBoolean(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG);

        if (!idempotenceEnabled && userConfiguredIdempotence && userConfiguredTransactions)
            throw new ConfigException("Cannot set a " + ProducerConfig.TRANSACTIONAL_ID_CONFIG + " without also enabling idempotence.");

        if (userConfiguredTransactions)
            idempotenceEnabled = true;

        if (idempotenceEnabled) {
            String transactionalId = config.getString(ProducerConfig.TRANSACTIONAL_ID_CONFIG);
            int transactionTimeoutMs = config.getInt(ProducerConfig.TRANSACTION_TIMEOUT_CONFIG);
            long retryBackoffMs = config.getLong(ProducerConfig.RETRY_BACKOFF_MS_CONFIG);
            transactionManager = new TransactionManager(logContext, transactionalId, transactionTimeoutMs, retryBackoffMs);
            if (transactionManager.isTransactional())
                log.info("Instantiated a transactional producer.");
            else
                log.info("Instantiated an idempotent producer.");
        }

        return transactionManager;
    }

    private static int configureRetries(ProducerConfig config, boolean idempotenceEnabled, Logger log) {
        boolean userConfiguredRetries = false;
        if (config.originals().containsKey(ProducerConfig.RETRIES_CONFIG)) {
            userConfiguredRetries = true;
        }
        if (idempotenceEnabled && !userConfiguredRetries) {
            // We recommend setting infinite retries when the idempotent producer is enabled, so it makes sense to make
            // this the default.
            log.info("Overriding the default retries config to the recommended value of {} since the idempotent " +
                    "producer is enabled.", Integer.MAX_VALUE);
            return Integer.MAX_VALUE;
        }
        if (idempotenceEnabled && config.getInt(ProducerConfig.RETRIES_CONFIG) == 0) {
            throw new ConfigException("Must set " + ProducerConfig.RETRIES_CONFIG + " to non-zero when using the idempotent producer.");
        }
        return config.getInt(ProducerConfig.RETRIES_CONFIG);
    }

    private static int configureInflightRequests(ProducerConfig config, boolean idempotenceEnabled) {
        if (idempotenceEnabled && 5 < config.getInt(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION)) {
            throw new ConfigException("Must set " + ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION + " to at most 5" +
                    " to use the idempotent producer.");
        }
        return config.getInt(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION);
    }

    private static short configureAcks(ProducerConfig config, boolean idempotenceEnabled, Logger log) {
        boolean userConfiguredAcks = false;
        short acks = (short) parseAcks(config.getString(ProducerConfig.ACKS_CONFIG));
        if (config.originals().containsKey(ProducerConfig.ACKS_CONFIG)) {
            userConfiguredAcks = true;
        }

        if (idempotenceEnabled && !userConfiguredAcks) {
            log.info("Overriding the default {} to all since idempotence is enabled.", ProducerConfig.ACKS_CONFIG);
            return -1;
        }

        if (idempotenceEnabled && acks != -1) {
            throw new ConfigException("Must set " + ProducerConfig.ACKS_CONFIG + " to all in order to use the idempotent " +
                    "producer. Otherwise we cannot guarantee idempotence.");
        }
        return acks;
    }

    private static int parseAcks(String acksString) {
        try {
            return acksString.trim().equalsIgnoreCase("all") ? -1 : Integer.parseInt(acksString.trim());
        } catch (NumberFormatException e) {
            throw new ConfigException("Invalid configuration value for 'acks': " + acksString);
        }
    }

    public void initTransactions() {
        throwIfNoTransactionManager();
        TransactionalRequestResult result = transactionManager.initializeTransactions();
        sender.wakeup();
        result.await();
    }

    public void beginTransaction() throws ProducerFencedException {
        throwIfNoTransactionManager();
        transactionManager.beginTransaction();
    }

    public void sendOffsetsToTransaction(Map<TopicPartition, OffsetAndMetadata> offsets,
                                         String consumerGroupId) throws ProducerFencedException {
        throwIfNoTransactionManager();
        TransactionalRequestResult result = transactionManager.sendOffsetsToTransaction(offsets, consumerGroupId);
        sender.wakeup();
        result.await();
    }

    public void commitTransaction() throws ProducerFencedException {
        throwIfNoTransactionManager();
        TransactionalRequestResult result = transactionManager.beginCommit();
        sender.wakeup();
        result.await();
    }

    public void abortTransaction() throws ProducerFencedException {
        throwIfNoTransactionManager();
        TransactionalRequestResult result = transactionManager.beginAbort();
        sender.wakeup();
        result.await();
    }

    @Override
    public Future<RecordMetadata> send(ProducerRecord<K, V> record) {
        return send(record, null);
    }

    @Override
    public Future<RecordMetadata> send(ProducerRecord<K, V> record, Callback callback) {
        // intercept the record, which can be potentially modified; this method does not throw exceptions
        ProducerRecord<K, V> interceptedRecord = this.interceptors.onSend(record);
        return doSend(interceptedRecord, callback);
    }

    private Future<RecordMetadata> doSend(ProducerRecord<K, V> record, Callback callback) {
        TopicPartition tp = null;
        try {
            // first make sure the metadata for the topic is available
            ClusterAndWaitTime clusterAndWaitTime = waitOnMetadata(record.topic(), record.partition(), maxBlockTimeMs);
            long remainingWaitMs = Math.max(0, maxBlockTimeMs - clusterAndWaitTime.waitedOnMetadataMs);
            Cluster cluster = clusterAndWaitTime.cluster;
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
            int partition = partition(record, serializedKey, serializedValue, cluster);
            tp = new TopicPartition(record.topic(), partition);

            setReadOnly(record.headers());
            Header[] headers = record.headers().toArray();

            int serializedSize = AbstractRecords.estimateSizeInBytesUpperBound(apiVersions.maxUsableProduceMagic(),
                    compressionType, serializedKey, serializedValue, headers);
            ensureValidRecordSize(serializedSize);
            long timestamp = record.timestamp() == null ? time.milliseconds() : record.timestamp();
            log.trace("Sending record {} with callback {} to topic {} partition {}", record, callback, record.topic(), partition);
            // producer callback will make sure to call both 'callback' and interceptor callback
            Callback interceptCallback = new InterceptorCallback<>(callback, this.interceptors, tp);

            if (transactionManager != null && transactionManager.isTransactional())
                transactionManager.maybeAddPartitionToTransaction(tp);

            RecordAccumulator.RecordAppendResult result = accumulator.append(tp, timestamp, serializedKey,
                    serializedValue, headers, interceptCallback, remainingWaitMs);
            if (result.batchIsFull || result.newBatchCreated) {
                log.trace("Waking up the sender since topic {} partition {} is either full or getting a new batch", record.topic(), partition);
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

    private void setReadOnly(Headers headers) {
        if (headers instanceof RecordHeaders) {
            ((RecordHeaders) headers).setReadOnly();
        }
    }

    private ClusterAndWaitTime waitOnMetadata(String topic, Integer partition, long maxWaitMs) throws InterruptedException {
        // add topic to metadata topic list if it is not there already and reset expiry
        metadata.add(topic);
        Cluster cluster = metadata.fetch();
        Integer partitionsCount = cluster.partitionCountForTopic(topic);
        // Return cached metadata if we have it, and if the record's partition is either undefined
        // or within the known partition range
        if (partitionsCount != null && (partition == null || partition < partitionsCount))
            return new ClusterAndWaitTime(cluster, 0);

        long begin = time.milliseconds();
        long remainingWaitMs = maxWaitMs;
        long elapsed;
        // Issue metadata requests until we have metadata for the topic or maxWaitTimeMs is exceeded.
        // In case we already have cached metadata for the topic, but the requested partition is greater
        // than expected, issue an update request only once. This is necessary in case the metadata
        // is stale and the number of partitions for this topic has increased in the meantime.
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

    /**
     * Validate that the record size isn't too large
     */
    private void ensureValidRecordSize(int size) {
        if (size > this.maxRequestSize)
            throw new RecordTooLargeException("The message is " + size +
                    " bytes when serialized which is larger than the maximum request size you have configured with the " +
                    ProducerConfig.MAX_REQUEST_SIZE_CONFIG +
                    " configuration.");
        if (size > this.totalMemorySize)
            throw new RecordTooLargeException("The message is " + size +
                    " bytes when serialized which is larger than the total memory buffer you have configured with the " +
                    ProducerConfig.BUFFER_MEMORY_CONFIG +
                    " configuration.");
    }

    @Override
    public void flush() {
        log.trace("Flushing accumulated records in producer.");
        this.accumulator.beginFlush();
        this.sender.wakeup();
        try {
            this.accumulator.awaitFlushCompletion();
        } catch (InterruptedException e) {
            throw new InterruptException("Flush interrupted.", e);
        }
    }

    @Override
    public List<PartitionInfo> partitionsFor(String topic) {
        Objects.requireNonNull(topic, "topic cannot be null");
        try {
            return waitOnMetadata(topic, null, maxBlockTimeMs).cluster.partitionsForTopic(topic);
        } catch (InterruptedException e) {
            throw new InterruptException(e);
        }
    }

    @Override
    public Map<MetricName, ? extends Metric> metrics() {
        return Collections.unmodifiableMap(this.metrics.metrics());
    }

    @Override
    public void close() {
        close(Long.MAX_VALUE, TimeUnit.MILLISECONDS);
    }

    @Override
    public void close(long timeout, TimeUnit timeUnit) {
        close(timeout, timeUnit, false);
    }

    private void close(long timeout, TimeUnit timeUnit, boolean swallowException) {
        if (timeout < 0)
            throw new IllegalArgumentException("The timeout cannot be negative.");

        log.info("Closing the Kafka producer with timeoutMillis = {} ms.", timeUnit.toMillis(timeout));
        // this will keep track of the first encountered exception
        AtomicReference<Throwable> firstException = new AtomicReference<>();
        boolean invokedFromCallback = Thread.currentThread() == this.ioThread;
        if (timeout > 0) {
            if (invokedFromCallback) {
                log.warn("Overriding close timeout {} ms to 0 ms in order to prevent useless blocking due to self-join. " +
                        "This means you have incorrectly invoked close with a non-zero timeout from the producer call-back.", timeout);
            } else {
                // Try to close gracefully.
                if (this.sender != null)
                    this.sender.initiateClose();
                if (this.ioThread != null) {
                    try {
                        this.ioThread.join(timeUnit.toMillis(timeout));
                    } catch (InterruptedException t) {
                        firstException.compareAndSet(null, new InterruptException(t));
                        log.error("Interrupted while joining ioThread", t);
                    }
                }
            }
        }

        if (this.sender != null && this.ioThread != null && this.ioThread.isAlive()) {
            log.info("Proceeding to force close the producer since pending requests could not be completed " +
                    "within timeout {} ms.", timeout);
            this.sender.forceClose();
            // Only join the sender thread when not calling from callback.
            if (!invokedFromCallback) {
                try {
                    this.ioThread.join();
                } catch (InterruptedException e) {
                    firstException.compareAndSet(null, new InterruptException(e));
                }
            }
        }

        ClientUtils.closeQuietly(interceptors, "producer interceptors", firstException);
        ClientUtils.closeQuietly(metrics, "producer metrics", firstException);
        ClientUtils.closeQuietly(keySerializer, "producer keySerializer", firstException);
        ClientUtils.closeQuietly(valueSerializer, "producer valueSerializer", firstException);
        ClientUtils.closeQuietly(partitioner, "producer partitioner", firstException);
        AppInfoParser.unregisterAppInfo(JMX_PREFIX, clientId, metrics);
        log.debug("Kafka producer has been closed");
        Throwable exception = firstException.get();
        if (exception != null && !swallowException) {
            if (exception instanceof InterruptException) {
                throw (InterruptException) exception;
            }
            throw new KafkaException("Failed to close kafka producer", exception);
        }
    }

    private ClusterResourceListeners configureClusterResourceListeners(Serializer<K> keySerializer, Serializer<V> valueSerializer, List<?>... candidateLists) {
        ClusterResourceListeners clusterResourceListeners = new ClusterResourceListeners();
        for (List<?> candidateList: candidateLists)
            clusterResourceListeners.maybeAddAll(candidateList);

        clusterResourceListeners.maybeAdd(keySerializer);
        clusterResourceListeners.maybeAdd(valueSerializer);
        return clusterResourceListeners;
    }

    private int partition(ProducerRecord<K, V> record, byte[] serializedKey, byte[] serializedValue, Cluster cluster) {
        Integer partition = record.partition();
        return partition != null ?
                partition :
                partitioner.partition(
                        record.topic(), record.key(), serializedKey, record.value(), serializedValue, cluster);
    }

    private void throwIfNoTransactionManager() {
        if (transactionManager == null)
            throw new IllegalStateException("Cannot use transactional methods without enabling transactions " +
                    "by setting the " + ProducerConfig.TRANSACTIONAL_ID_CONFIG + " configuration property");
    }

    private static class ClusterAndWaitTime {
        final Cluster cluster;
        final long waitedOnMetadataMs;
        ClusterAndWaitTime(Cluster cluster, long waitedOnMetadataMs) {
            this.cluster = cluster;
            this.waitedOnMetadataMs = waitedOnMetadataMs;
        }
    }

    private static class FutureFailure implements Future<RecordMetadata> {

        private final ExecutionException exception;

        public FutureFailure(Exception exception) {
            this.exception = new ExecutionException(exception);
        }

        @Override
        public boolean cancel(boolean interrupt) {
            return false;
        }

        @Override
        public RecordMetadata get() throws ExecutionException {
            throw this.exception;
        }

        @Override
        public RecordMetadata get(long timeout, TimeUnit unit) throws ExecutionException {
            throw this.exception;
        }

        @Override
        public boolean isCancelled() {
            return false;
        }

        @Override
        public boolean isDone() {
            return true;
        }

    }

    private static class InterceptorCallback<K, V> implements Callback {
        private final Callback userCallback;
        private final ProducerInterceptors<K, V> interceptors;
        private final TopicPartition tp;

        private InterceptorCallback(Callback userCallback, ProducerInterceptors<K, V> interceptors, TopicPartition tp) {
            this.userCallback = userCallback;
            this.interceptors = interceptors;
            this.tp = tp;
        }

        public void onCompletion(RecordMetadata metadata, Exception exception) {
            metadata = metadata != null ? metadata : new RecordMetadata(tp, -1, -1, RecordBatch.NO_TIMESTAMP, Long.valueOf(-1L), -1, -1);
            this.interceptors.onAcknowledgement(metadata, exception);
            if (this.userCallback != null)
                this.userCallback.onCompletion(metadata, exception);
        }
    }
}

```
