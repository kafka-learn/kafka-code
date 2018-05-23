---

title: "Producer接口"
date: 2018-5-22 11:48

---

[TOC]

# Producer接口的实现

```java
/*
 * Licensed to the Apache Software Foundation (ASF) under one or more
 * contributor license agreements. See the NOTICE file distributed with
 * this work for additional information regarding copyright ownership.
 * The ASF licenses this file to You under the Apache License, Version 2.0
 * (the "License"); you may not use this file except in compliance with
 * the License. You may obtain a copy of the License at
 *
 *    http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package org.apache.kafka.clients.producer;

import org.apache.kafka.common.Metric;
import org.apache.kafka.common.MetricName;
import org.apache.kafka.common.PartitionInfo;
import org.apache.kafka.clients.consumer.OffsetAndMetadata;
import org.apache.kafka.common.TopicPartition;
import org.apache.kafka.common.errors.ProducerFencedException;

import java.io.Closeable;
import java.util.List;
import java.util.Map;
import java.util.concurrent.Future;
import java.util.concurrent.TimeUnit;

/**
 * The interface for the {@link KafkaProducer}
 * @see KafkaProducer
 * @see MockProducer
 */
public interface Producer<K, V> extends Closeable {

    /**
     * See {@link KafkaProducer#initTransactions()}
     */
    void initTransactions();

    /**
     * See {@link KafkaProducer#beginTransaction()}
     */
    void beginTransaction() throws ProducerFencedException;

    /**
     * See {@link KafkaProducer#sendOffsetsToTransaction(Map, String)}
     */
    void sendOffsetsToTransaction(Map<TopicPartition, OffsetAndMetadata> offsets,
                                  String consumerGroupId) throws ProducerFencedException;

    /**
     * See {@link KafkaProducer#commitTransaction()}
     */
    void commitTransaction() throws ProducerFencedException;

    /**
     * See {@link KafkaProducer#abortTransaction()}
     */
    void abortTransaction() throws ProducerFencedException;

    /**
     * See {@link KafkaProducer#send(ProducerRecord)}
     */
    Future<RecordMetadata> send(ProducerRecord<K, V> record);

    /**
     * See {@link KafkaProducer#send(ProducerRecord, Callback)}
     */
    Future<RecordMetadata> send(ProducerRecord<K, V> record, Callback callback);

    /**
     * See {@link KafkaProducer#flush()}
     */
    void flush();

    /**
     * See {@link KafkaProducer#partitionsFor(String)}
     */
    List<PartitionInfo> partitionsFor(String topic);

    /**
     * See {@link KafkaProducer#metrics()}
     */
    Map<MetricName, ? extends Metric> metrics();

    /**
     * See {@link KafkaProducer#close()}
     */
    void close();

    /**
     * See {@link KafkaProducer#close(long, TimeUnit)}
     */
    void close(long timeout, TimeUnit unit);

}

```

Producer接口中主要定义了为KafkaProducer使用的API，分为以下几类方法（为描述方便，暂时省略以下方法的参数）：

1. ```transaction```方法，具体分为```initTransactions()```, ```beginTransaction()```, ```sendOffsetsToTransaction()```, ```commitTransaction()```和```abortTransaction()```。主要作用是对Producer进行事务管理。

2. ```send()```方法，发送消息到RecordAccumulator中暂存，提供了同步和异步两套方案。

3. ```flush()```方法，调用该方法将立即发送所有缓冲的消息，并将阻塞其他与这些消息相关的请求。

4. ```partitionsFor()```方法，用于获取metadata中指定topic的分区信息。

5. ```metrics()```方法，用于获取生产者维护的完整的内部统计信息。

6. ```close()```方法，关闭这个生产者。在所有发送的请求完成之前，该方法会阻塞。