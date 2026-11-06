← [Back to Index](../README.md)

# Apache Kafka vs Google Cloud Pub/Sub: Architecture & System Design Interview Guide

A comprehensive, production-grade comparative analysis of **Apache Kafka** and **Google Cloud Pub/Sub** for backend engineers, system architects, and technical interview preparation.

---

## 1. High-Level Overview & Core Philosophy

| Dimension | Apache Kafka | Google Cloud Pub/Sub |
| :--- | :--- | :--- |
| **Fundamental Model** | Distributed, partitioned, immutable **append-only commit log**. | Fully-managed, serverless, globally distributed **publish/subscribe message broker**. |
| **Consumption Paradigm** | **Pull-based** streaming. Consumers pull batches from partitions and track their own log offsets. | **Push or Pull** (StreamingPull) asynchronous queue model with server-side ack tracking. |
| **Primary Use Cases** | High-throughput event streaming, event sourcing, stream processing (Kafka Streams, Apache Flink), long-term log retention, replayable event stores. | Cloud-native event distribution, microservices choreography, asynchronous background processing, serverless triggers (Cloud Functions, Cloud Run). |
| **Infrastructure Model** | Infrastructure-centric (Brokers, clusters, disks, JVM tuning, ZooKeeper/KRaft). Self-hosted or semi-managed (Confluent Cloud, Amazon MSK). | 100% serverless, zero-ops, global endpoint, automated elastic horizontal scaling. |
| **Replayability** | **Native first-class citizen**. Events are retained regardless of consumption; consumers can arbitrarily seek offsets back in time. | **Secondary capability**. Supports Seek via snapshots or timestamps within retention windows, but fundamentally oriented around stateful message acknowledgment and removal. |

### Core Architectural Mindset
* **Apache Kafka** treats data as an **ordered sequence of historical events (the log)**. A topic is split into physical partitions hosted on specific broker disks. Consumers act as read pointers over immutable logs.
* **Google Cloud Pub/Sub** treats data as **discrete ephemerally held messages**. Publishers send messages to a global topic, and subscriptions distribute copies to subscribers. It abstracts away partitions, servers, and storage mechanics behind a global managed mesh.

---

## 2. Architecture Deep-Dive

### 2.1 Apache Kafka Architecture

Kafka is structured around physical log partitions distributed across a cluster of broker nodes.

```
                           +-------------------------------------------------+
                           |                 KAFKA CLUSTER                   |
                           |                                                 |
  +---------------+        |  +-------------------------------------------+  |        +-------------------+
  |   Producer    |        |  |  Broker 1                                 |  |        |  Consumer Group A |
  | (Partitioner) |--Key A--->|  [Topic 1 - Partition 0 (Leader)]         |==========>|  Consumer 1       |
  |               |        |  |  [Topic 1 - Partition 1 (Follower)]       |  |        +-------------------+
  +---------------+        |  +-------------------------------------------+  |
          |                |                        ^                        |
          |                |           Replication  | Sync                   |        +-------------------+
          |                |                        v                        |        |  Consumer Group B |
          |                |  +-------------------------------------------+  |        |  Consumer 1       |
          +----------Key B--->|  Broker 2                                 |==========>|  (Reads Part 0)   |
                           |  |  [Topic 1 - Partition 0 (Follower)]       |  |        |                   |
                           |  |  [Topic 1 - Partition 1 (Leader)]         |==========>|  Consumer 2       |
                           |  +-------------------------------------------+  |        |  (Reads Part 1)   |
                           |                                                 |        +-------------------+
                           |  Metadata Quorum: KRaft (or legacy ZooKeeper)   |
                           +-------------------------------------------------+
```

#### Key Kafka Components:
1. **Brokers**: Stateless-leaning JVM daemons that serve producer and consumer requests and persist partition logs to disk using the OS page cache and zero-copy transfers (`sendfile`).
2. **Metadata Quorum (ZooKeeper vs KRaft)**:
   * **Legacy (ZooKeeper)**: Handled controller election, topic metadata, broker liveness, and ACLs. Scalability hit a bottleneck around ~200k partitions per cluster; controller failover took tens of seconds.
   * **Modern (KRaft - KIP-500)**: Integrated Raft consensus protocol running on dedicated or shared controller brokers. Metadata is stored in a replicated internal topic (`@metadata`). Provides sub-second controller failover and scales to millions of partitions.
3. **Topics & Partitions**:
   * A **Topic** is a logical category.
   * A **Partition** is the physical unit of parallelism, ordering, and storage. It is an ordered, immutable sequence of records continuously appended.
4. **Offsets**:
   * Every message in a partition receives a sequential 64-bit integer identifier called an **offset**.
   * Kafka maintains the consumer's current position via internal topic `__consumer_offsets`.
5. **Replication & ISR (In-Sync Replicas)**:
   * Each partition has 1 **Leader** and $N-1$ **Followers**.
   * Only the Leader serves read and write requests (unless rack-aware follower fetching is configured).
   * **ISR**: Set of replicas keeping up with the leader within `replica.lag.time.max.ms`.
   * Write acknowledgment is governed by `acks` (`0`, `1`, `all` / `-1`) and `min.insync.replicas`.
6. **Consumer Groups**:
   * Set of consumers cooperating to consume data from a topic.
   * **Rule of Partition Assignment**: Each partition is assigned to exactly **one** consumer instance within a group at any given time.
   * Group coordination and partition assignment are governed by the Group Coordinator broker and Rebalance Protocols (Eager vs Cooperative Sticky).

---

### 2.2 Google Cloud Pub/Sub Architecture

Pub/Sub splits control and data planes across a globally distributed, serverless architecture that decouples routing, storage, and message delivery.

```
                           +-----------------------------------------------------------------------+
                           |                        GOOGLE CLOUD PUB/SUB                           |
                           |                                                                       |
  +---------------+        |     +--------------------+          +---------------------------+     |        +-------------------+
  |   Publisher   |--HTTPS/----->|  Forwarder Layer   |--------->| Storage Layer             |     |        |   Subscriber 1    |
  |               |  gRPC  |     |  (Global Ingress)  |          | (Colossus / Spanner / SSD)|     |  Pull  | (Receives msg A,  |
  +---------------+        |     +--------------------+          +---------------------------+     |<======>|  Dynamic share)   |
                           |               |                                   |                   | (gRPC) +-------------------+
                           |               v                                   v                   |
                           |     +-----------------------------------------------------------+     |        +-------------------+
                           |     |                       Router Layer                        |====Push=====>|   Subscriber 2    |
                           |     | (Tracks Subscriptions, ACKs, Leases, Ack Deadline Timers) |     | (HTTP) | (Cloud Run / App) |
                           |     +-----------------------------------------------------------+     |        +-------------------+
                           +-----------------------------------------------------------------------+
```

#### Key Pub/Sub Components:
1. **Forwarder / Ingress Layer**:
   * Proxies located across Google's edge locations worldwide. Handles authentication, rate limiting, and routes published messages to the nearest storage proxies.
2. **Storage Layer**:
   * Backed by Google's distributed file system (**Colossus**) and internal consensus engines (**Spanner** / Chubby). Writes are replicated across multiple zones before an acknowledgment is returned to the publisher.
3. **Router Layer**:
   * Dispatches messages from storage to active subscribers.
   * Tracks **unacknowledged messages**, lease renewals, ACK deadlines, and delivery retry backoffs.
4. **Topics & Subscriptions**:
   * **Topic**: A named resource to which messages are sent.
   * **Subscription**: A named resource representing the stream of messages from a single, specific topic to be delivered to the subscribing application. Multiple subscriptions to the same topic create a fan-out (1:N) pattern.
5. **Push vs Pull Delivery**:
   * **Pull (StreamingPull)**: Bidirectional gRPC stream. High throughput, low latency; subscribers request messages and send ACKs over persistent TCP streams.
   * **Push**: Pub/Sub initiates an HTTP POST request to an exposed webhook HTTPS endpoint. Ideal for serverless targets (Cloud Run, Cloud Functions).
6. **Ack Deadline & Lease Management**:
   * When a subscriber receives a message, Pub/Sub starts an **Ack Deadline** countdown (default 10s, configurable up to 600s).
   * If the subscriber does not ACK or NACK within the deadline, Pub/Sub assumes consumer failure and redelivers the message.
   * The client SDK runs a background thread called the **Automatic Lease Extender** (`modifyAckDeadline`) to keep long-running tasks alive.

---

## 3. Deep Architectural Comparisons

### 3.1 Partitions vs Subscriptions & Scaling Model

This is the most critical conceptual difference tested in system design interviews.

```
KAFKA: Rigid Partition Coupling                 PUB/SUB: Dynamic Fluid Distribution

  Topic: Orders (3 Partitions)                    Topic: Orders (Logical Stream)
  +--------------+                                +-----------------------------+
  | Partition 0  | ===> Consumer 1                |      Shared Message Pool    |
  +--------------+                                +-----------------------------+
  | Partition 1  | ===> Consumer 2                                |
  +--------------+                                +---------------+---------------+
  | Partition 2  | ===> Consumer 3                |               |               |
  +--------------+                              Worker 1       Worker 2       Worker 3
                                                (Dynamically pulls any available message,
  Max Active Consumers = 3                       Scales up to thousands of workers without
  Consumer 4 would be IDLE!                      repartitioning or rebalancing)
```

#### The Kafka Model:
* Partition count is a **hard upper bound** on consumer concurrency within a single consumer group. If you have 10 partitions, having 12 consumers in a consumer group means 2 consumers sit 100% idle.
* Adding partitions requires careful planning: hashing algorithms may break key distribution, consumer group rebalancing is triggered, and partition counts cannot be easily decreased.
* **Pro**: Deterministic routing, low overhead, strictly ordered per partition, high cache locality.
* **Con**: Dynamic autoscaling is constrained. Spiky workloads require pre-provisioning excess partitions.

#### The Pub/Sub Model:
* Messages are not physically locked into static partitions. They reside in a dynamic storage pool tied to a Subscription.
* If a traffic spike occurs, you can scale subscribers from 5 to 500 instances instantly. Each instance pulls available unleased messages concurrently.
* **Pro**: True elastic horizontal autoscaling. Work-stealing prevents slow consumers from blocking independent messages.
* **Con**: High router coordination overhead, non-deterministic delivery order (without explicit ordering keys), per-message ACK tracking cost.

---

### 3.2 Delivery Guarantees: At-Least-Once & Exactly-Once

Both systems default to **At-Least-Once** delivery. Duplicate messages can and will occur during network partitions, timeouts, and process crashes.

```
+---------------------------------------------------------------------------------------------------+
|                                 EXACTLY-ONCE SEMANTICS (EOS)                                     |
+------------------------------------+--------------------------------------------------------------+
|            Apache Kafka            |                    Google Cloud Pub/Sub                      |
+------------------------------------+--------------------------------------------------------------+
| 1. Idempotent Producer:            | 1. Exactly-Once Subscriptions:                               |
|    - Producer assigns PID and      |    - Enabled at the subscription level.                      |
|      monotonically increasing      |    - Guarantees that if a message is acknowledged within its |
|      Sequence Numbers per batch.   |      ack deadline, it will not be redelivered.               |
|    - Broker deduplicates at log.   |                                                              |
|                                    | 2. Ack ID Locking:                                           |
| 2. Two-Phase Commit Transactions:  |    - Pub/Sub tracks message ack status globally. If an ACK   |
|    - Atomic writes across multiple |      succeeds, retried sends or duplicate fetches are        |
|      topics and partitions.        |      dropped.                                                |
|    - Atomically commits read       |                                                              |
|      offsets (__consumer_offsets). | 3. Limitation:                                               |
|    - Read committed isolation mode |    - Failures during processing prior to ACK redeliver;      |
|      on downstream consumers.      |      end-to-end downstream side effects still require        |
|                                    |      application idempotency.                                |
+------------------------------------+--------------------------------------------------------------+
```

#### Kafka Exactly-Once Details:
* Works seamlessly across **Read-Process-Write** loops (e.g., Kafka Streams, Flink).
* Transaction Coordinator assigns a `TransactionalId` and registers transactional markers in partition logs.
* Downstream consumers configured with `isolation.level = "read_committed"` buffer uncommitted messages and only emit records when the commit marker appears.

#### Pub/Sub Exactly-Once Details:
* Pub/Sub offers **Exactly-Once Delivery** at the subscription level.
* When enabled, if an ACK is acknowledged by the router, subsequent lease expirations or network blips will not trigger redelivery.
* Pub/Sub guarantees no duplicate deliveries *after* a successful ack within the ack deadline. However, if your subscriber crashes halfway through processing, the message will still be redelivered.

---

### 3.3 Message Ordering

| Strategy | Apache Kafka | Google Cloud Pub/Sub |
| :--- | :--- | :--- |
| **Default Behavior** | Guaranteed per partition. | Out-of-order delivery across all subscribers. |
| **Ordering Mechanism** | Record **Key**. Kafka computes `murmur2(key) % numPartitions`. All records with the same key go to the exact same partition. | **Ordering Key**. Publishers attach a string `ordering_key` to messages. |
| **Concurrency Impact** | Limited by partition count. All keys on a partition are processed sequentially by one consumer. | Dynamic per key. Pub/Sub routes messages with the same ordering key to the same subscriber in sequence. Different keys scale independently. |
| **Failure Mode** | If a consumer fails, processing pauses for that entire partition during rebalance. | If an ordered message fails/NACKs, **all subsequent messages for that ordering key are blocked** until the failed message is ACKed or expires. |

---

### 3.4 Storage, Retention & Replayability

```
KAFKA: Immutable Commit Log (Time / Compacted)
[Offset 0][Offset 1][Offset 2][Offset 3][Offset 4][Offset 5][Offset 6] ... [Offset N]
   ^                                  ^                                       ^
   |                                  |                                       |
 Earliest Retention           Consumer A Offset                       Producer Head
 (Replay by seeking to Offset 0)

PUB/SUB: Ephemeral State Machine with Snapshot/Seek
[Published Message] ---> [Undelivered / Leased Queue] ---> [ACK Received] ---> [Discarded]
                                   |
                  (If Ack Deadline expires -> Redeliver)
```

* **Kafka**: Designed as a durable, long-term event store.
  * Retention configured by time (`log.retention.hours = 168`) or size (`log.retention.bytes`).
  * Supports **Log Compaction** (`cleanup.policy = compact`): retains the latest known value for each key forever (ideal for changelogs, KTables, CDC).
  * Consumer replay is trivial: call `consumer.seek(new TopicPartition("orders", 0), targetOffset)`.
* **Pub/Sub**: Designed as an asynchronous transit pipeline.
  * Stores unacknowledged messages up to 7 days (or 31 days with message retention duration).
  * Once a message is ACKed across all subscriptions, it is flagged for deletion.
  * **Seek Feature**: Pub/Sub allows you to create a snapshot of a subscription or seek back to a timestamp within the retention window. However, it alters the subscription state for **all** subscribers sharing that subscription, rather than a single consumer instance.

---

### 3.5 Throughput, Latency & Performance Mechanics

#### Why Kafka achieves staggering throughput ($10^6+$ msg/sec per broker):
1. **OS Page Cache**: Kafka does not cache messages in the JVM heap; it writes to the OS filesystem cache directly, avoiding JVM GC overhead.
2. **Sequential Disk I/O**: Kafka only appends to the end of segment files (`.log`). Sequential disk writes match or exceed random memory access speeds.
3. **Zero-Copy Data Transfer (`sendfile`)**: Data moves directly from the OS page cache to the network socket buffer without copying into user-space memory:
   ```
   Disk File -> OS Page Cache ---[sendfile syscall]---> NIC Socket Buffer -> Network
   ```
4. **End-to-End Batching**: Producers compress (Snappy, LZ4, Zstandard) and send batches of records; brokers write batches intact without uncompressing; consumers read batches directly.

#### Pub/Sub Throughput & Latency Profile:
* Global multi-tenant cloud service.
* Typical publish latency: $20\text{ ms} - 50\text{ ms}$ (due to multi-zone replication, routing hops, and HTTPS/gRPC gateway overhead).
* Massive horizontal scalability: Automatically absorbs traffic spikes from $100$ to $10,000,000$ messages/sec without capacity provisioning, but per-partition microsecond latency is impossible.

---

## 4. Head-to-Head Comparison Table

| Feature / Dimension | Apache Kafka | Google Cloud Pub/Sub |
| :--- | :--- | :--- |
| **Architecture** | Distributed Commit Log | Message Broker / Router Mesh |
| **Hosting Model** | Self-managed (EC2, K8s) or Managed (MSK, Confluent) | Fully-managed Serverless SaaS (Google Cloud) |
| **Unit of Partitioning** | Topic Partition | Virtual / Ordering Key |
| **Consumer Scaling Limit** | $\le \text{Number of Partitions}$ | Virtually unlimited subscribers |
| **Consumption Style** | Pull (batch poll) | Pull (StreamingPull) or Push (HTTP POST) |
| **Consumer Tracking** | Offset integer stored in `__consumer_offsets` | Ack ID leased per message, tracked server-side |
| **Ordering Guarantee** | Strict per partition | Out of order by default; strict per `ordering_key` |
| **Latency** | Sub-10ms (p99 $\approx 2-5\text{ ms}$) | $20 - 60\text{ ms}$ |
| **Throughput / Node** | Very high ($>1\text{ GB/sec}$ aggregate per cluster) | Scaled automatically by Google quota system |
| **Message Size Limit** | Configurable (default $1\text{ MB}$, can tune to $10-20\text{ MB}$) | $10\text{ MB}$ hard limit |
| **Retention Policy** | Milliseconds to infinite (time or size-based) | Up to 7 days (or 31 days with message retention) |
| **Log Compaction** | Supported natively (Key-value state stores) | Not supported |
| **Backpressure** | Consumer-controlled (Consumer stops pulling) | Client-side flow control / ack deadline extension |
| **Dead Letter Queue** | Application-managed or Kafka Connect / Camel DLQ | First-class native feature (`deadLetterPolicy`) |
| **Push Webhooks** | Requires proxy / connector (Kafka REST Proxy) | Built-in native Push subscriptions |
| **Ecosystem** | Kafka Streams, Flink, Spark, Debezium, Connect | Cloud Dataflow (Apache Beam), BigQuery, Cloud Run |
| **Pricing Model** | Infrastructure-based (Broker VMs, storage, networking) | Pay-per-use (Ingress data volume, storage retention) |

---

## 5. When to Use Which: Decision Framework

```
                                      [Messaging Requirement]
                                                 |
                       +-------------------------+-------------------------+
                       |                                                   |
              Need Event Sourcing,                               Need Serverless, Zero-Ops,
              Stream Processing, or                             Simple Event Fan-Out, or
             Long-Term Log Replay?                              Push to Cloud Functions?
                       |                                                   |
                       v                                                   v
             [Choose Apache Kafka]                               [Choose Cloud Pub/Sub]
                       |                                                   |
         +-------------+-------------+                       +-------------+-------------+
         |                           |                       |                           |
    Ultra-low latency          Rich stream              GCP-native stack       Unpredictable
    (sub-10ms, financial)      frameworks               (Cloud Run/Dataflow)   spiky workloads
```

### Choose Apache Kafka when:
1. **You need stream processing engines**: Heavy usage of Kafka Streams, Apache Flink, Apache Spark Streaming, or RocksDB-backed stateful transformations.
2. **Event Sourcing & Log Replay**: Downstream services frequently rebuild state, spin up new microservices that replay years of historical transactions, or use log compaction for key-value snapshots.
3. **Extreme Throughput with Low Latency**: Hundreds of thousands of events per second with strict sub-10ms p99 latency SLA requirements.
4. **Cloud-Agnostic / Multi-Cloud / Hybrid**: Must run on AWS, GCP, Azure, or on-premises Kubernetes without proprietary cloud lock-in.
5. **Change Data Capture (CDC)**: Ingesting database binlogs via Debezium directly into Kafka topics.

### Choose Google Cloud Pub/Sub when:
1. **You want Zero Infrastructure Management (Zero-Ops)**: No desire to manage brokers, KRaft controllers, partition rebalancing, storage sizing, or JVM garbage collection.
2. **Highly Elastic or Unpredictable Traffic**: Workloads that swing from 0 to 1,000,000 requests/sec where auto-scaling must happen instantaneously without cluster repartitioning.
3. **Serverless Microservice Workflows**: Pushing events to Google Cloud Run, Google Cloud Functions, App Engine, or external HTTPS webhooks without polling daemons.
4. **Independent Parallel Consumers**: You have many heterogeneous workers that process messages at variable speeds, and you want dynamic work-stealing without partition blocking.
5. **GCP Native Integration**: Ingesting directly into BigQuery (BigQuery subscription) or Cloud Storage with zero custom integration code.

---

## 6. Production Java Code Examples

### 6.1 Apache Kafka: Idempotent Producer & Manual Offset Consumer

#### Dependencies (`pom.xml`):
```xml
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-clients</artifactId>
    <version>3.7.0</version>
</dependency>
```

#### Kafka Producer (Idempotent, Safe, Production-Grade):
```java
package com.example.kafka;

import org.apache.kafka.clients.producer.*;
import org.apache.kafka.common.serialization.StringSerializer;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.time.Duration;
import java.util.Properties;

public class ProductionKafkaProducer {
    private static final Logger log = LoggerFactory.getLogger(ProductionKafkaProducer.class);
    private final Producer<String, String> producer;

    public ProductionKafkaProducer(String bootstrapServers) {
        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());

        // Reliability & Exactly-Once configuration
        props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, "true"); // enforces acks=all, retries=MAX, max.in.flight<=5
        props.put(ProducerConfig.ACKS_CONFIG, "all");
        props.put(ProducerConfig.RETRIES_CONFIG, Integer.MAX_VALUE);
        props.put(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, "5");

        // High-throughput batching & compression
        props.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "snappy");
        props.put(ProducerConfig.LINGER_MS_CONFIG, "20");
        props.put(ProducerConfig.BATCH_SIZE_CONFIG, Integer.toString(32 * 1024)); // 32 KB batch

        this.producer = new KafkaProducer<>(props);
    }

    public void sendMessage(String topic, String key, String value) {
        ProducerRecord<String, String> record = new ProducerRecord<>(topic, key, value);
        
        producer.send(record, new Callback() {
            @Override
            public void onCompletion(RecordMetadata metadata, Exception exception) {
                if (exception == null) {
                    log.info("Produced to topic={}, partition={}, offset={}, timestamp={}",
                            metadata.topic(), metadata.partition(), metadata.offset(), metadata.timestamp());
                } else {
                    log.error("Failed to produce message with key={}", key, exception);
                }
            }
        });
    }

    public void close() {
        producer.flush();
        producer.close(Duration.ofSeconds(10));
    }
}
```

#### Kafka Consumer (Manual Sync/Async Commit & Graceful Shutdown):
```java
package com.example.kafka;

import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.TopicPartition;
import org.apache.kafka.common.errors.WakeupException;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.time.Duration;
import java.util.*;

public class ProductionKafkaConsumer implements Runnable {
    private static final Logger log = LoggerFactory.getLogger(ProductionKafkaConsumer.class);
    private final KafkaConsumer<String, String> consumer;
    private final String topic;
    private volatile boolean running = true;

    public ProductionKafkaConsumer(String bootstrapServers, String groupId, String topic) {
        this.topic = topic;
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        props.put(ConsumerConfig.GROUP_ID_CONFIG, groupId);
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());

        // Commit control & Rebalance tuning
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false"); // Manual commit for at-least-once control
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, "500");
        props.put(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG, "300000"); // 5 minutes processing timeout
        props.put(ConsumerConfig.PARTITION_ASSIGNMENT_STRATEGY_CONFIG, 
                CooperativeStickyAssignor.class.getName()); // Non-stop-the-world rebalancing

        this.consumer = new KafkaConsumer<>(props);
    }

    @Override
    public void run() {
        try {
            consumer.subscribe(Collections.singletonList(topic));
            while (running) {
                ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(500));
                if (records.isEmpty()) continue;

                for (ConsumerRecord<String, String> record : records) {
                    processRecord(record);
                }

                // Synchronous offset commit after batch processing
                consumer.commitSync();
            }
        } catch (WakeupException e) {
            if (running) throw e;
        } catch (Exception e) {
            log.error("Unexpected error in consumer loop", e);
        } finally {
            try {
                consumer.commitSync(); // Final commit before closing
            } finally {
                consumer.close();
                log.info("Consumer closed successfully.");
            }
        }
    }

    private void processRecord(ConsumerRecord<String, String> record) {
        log.info("Consumed record key={}, value={}, partition={}, offset={}",
                record.key(), record.value(), record.partition(), record.offset());
    }

    public void shutdown() {
        running = false;
        consumer.wakeup(); // Thread-safe trigger to wake consumer out of poll()
    }
}
```

---

### 6.2 Google Cloud Pub/Sub: Publisher with Ordering & Async Subscriber

#### Dependencies (`pom.xml`):
```xml
<dependency>
    <groupId>com.google.cloud</groupId>
    <artifactId>google-cloud-pubsub</artifactId>
    <version>1.129.0</version>
</dependency>
```

#### Pub/Sub Publisher (Batching & Ordering Key):
```java
package com.example.pubsub;

import com.google.api.core.ApiFuture;
import com.google.api.core.ApiFutureCallback;
import com.google.api.core.ApiFutures;
import com.google.api.gax.batching.BatchingSettings;
import com.google.api.gax.rpc.ApiException;
import com.google.cloud.pubsub.v1.Publisher;
import com.google.common.util.concurrent.MoreExecutors;
import com.google.protobuf.ByteString;
import com.google.pubsub.v1.PubsubMessage;
import com.google.pubsub.v1.TopicName;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.threeten.bp.Duration;

import java.io.IOException;
import java.util.concurrent.TimeUnit;

public class ProductionPubSubPublisher {
    private static final Logger log = LoggerFactory.getLogger(ProductionPubSubPublisher.class);
    private final Publisher publisher;

    public ProductionPubSubPublisher(String projectId, String topicId) throws IOException {
        TopicName topicName = TopicName.of(projectId, topicId);

        // High-throughput batching settings
        BatchingSettings batchingSettings = BatchingSettings.newBuilder()
                .setElementCountThreshold(100L)
                .setRequestByteThreshold(1024L * 1024L) // 1 MB
                .setDelayThreshold(Duration.ofMillis(10))
                .build();

        this.publisher = Publisher.newBuilder(topicName)
                .setBatchingSettings(batchingSettings)
                .setEnableMessageOrdering(true) // Crucial if ordering keys are used
                .build();
    }

    public void publishWithOrdering(String orderingKey, String payload) {
        ByteString data = ByteString.copyFromUtf8(payload);
        PubsubMessage pubsubMessage = PubsubMessage.newBuilder()
                .setData(data)
                .setOrderingKey(orderingKey) // Guarantees in-order delivery for identical keys
                .putAttributes("event_type", "ORDER_CREATED")
                .build();

        ApiFuture<String> future = publisher.publish(pubsubMessage);

        ApiFutures.addCallback(future, new ApiFutureCallback<String>() {
            @Override
            public void onSuccess(String messageId) {
                log.info("Published message ID={} with key={}", messageId, orderingKey);
            }

            @Override
            public void onFailure(Throwable throwable) {
                if (throwable instanceof ApiException) {
                    ApiException apiException = (ApiException) throwable;
                    log.error("PubSub API failure code={}: {}", apiException.getStatusCode().getCode(), apiException.getMessage());
                } else {
                    log.error("Failed to publish message for key={}", orderingKey, throwable);
                }
            }
        }, MoreExecutors.directExecutor());
    }

    public void shutdown() throws Exception {
        publisher.shutdown();
        publisher.awaitTermination(1, TimeUnit.MINUTES);
    }
}
```

#### Pub/Sub Asynchronous Subscriber (StreamingPull with Flow Control):
```java
package com.example.pubsub;

import com.google.api.gax.batching.FlowControlSettings;
import com.google.cloud.pubsub.v1.AckReplyConsumer;
import com.google.cloud.pubsub.v1.MessageReceiver;
import com.google.cloud.pubsub.v1.Subscriber;
import com.google.pubsub.v1.ProjectSubscriptionName;
import com.google.pubsub.v1.PubsubMessage;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.threeten.bp.Duration;

public class ProductionPubSubSubscriber {
    private static final Logger log = LoggerFactory.getLogger(ProductionPubSubSubscriber.class);
    private final Subscriber subscriber;

    public ProductionPubSubSubscriber(String projectId, String subscriptionId) {
        ProjectSubscriptionName subscriptionName = ProjectSubscriptionName.of(projectId, subscriptionId);

        MessageReceiver receiver = new MessageReceiver() {
            @Override
            public void receiveMessage(PubsubMessage message, AckReplyConsumer consumer) {
                try {
                    String payload = message.getData().toStringUtf8();
                    String orderingKey = message.getOrderingKey();
                    log.info("Received message ID={}, key={}, data={}", message.getMessageId(), orderingKey, payload);

                    // Execute business logic
                    processBusinessLogic(payload);

                    // Acknowledge receipt
                    consumer.ack();
                } catch (Exception e) {
                    log.error("Failed processing message ID={}, sending NACK", message.getMessageId(), e);
                    // Negative ACK: signals Pub/Sub to redeliver immediately or back off
                    consumer.nack();
                }
            }
        };

        // Client-side Flow Control to prevent OOM
        FlowControlSettings flowControlSettings = FlowControlSettings.newBuilder()
                .setMaxOutstandingElementCount(1000L) // Max 1000 messages buffered in memory
                .setMaxOutstandingRequestBytes(50L * 1024L * 1024L) // Max 50 MB
                .build();

        this.subscriber = Subscriber.newBuilder(subscriptionName, receiver)
                .setFlowControlSettings(flowControlSettings)
                .setMaxAckExtensionPeriod(Duration.ofMinutes(10)) // Auto-extend deadline up to 10 min
                .build();
    }

    private void processBusinessLogic(String payload) {
        // Business implementation...
    }

    public void start() {
        subscriber.startAsync().awaitRunning();
        log.info("Pub/Sub Subscriber started listening...");
    }

    public void stop() {
        subscriber.stopAsync().awaitTerminated();
        log.info("Pub/Sub Subscriber stopped.");
    }
}
```

---

## 7. Common Interview Questions & In-Depth Answers

### Q1: "How does Kafka achieve higher throughput than traditional message brokers or Pub/Sub on single instances?"
**Answer Focus**:
Interviewers look for lower-level OS and storage mechanics:
1. **OS Page Cache & Sequential I/O**: Kafka bypasses JVM heap caches for raw OS page cache. Append-only sequential disk writes eliminate random disk head seek latency, performing similarly to RAM.
2. **Zero-Copy Architecture**: Uses the Linux `sendfile(2)` system call. Data transfers directly from OS Page Cache $\to$ Socket Buffer $\to$ NIC without crossing the kernel-user space boundary twice.
3. **End-to-End Batching & Compression**: Producers accumulate messages into memory buffers based on `linger.ms` and `batch.size`, compress them once, write them untouched to disk, and stream them compressed over the network directly to consumers.
4. **Log Partitioning**: Partitions act as independent shards. Writes scale linearly across physical disks across multiple brokers.

---

### Q2: "What happens when a Kafka consumer crashes versus when a Pub/Sub subscriber crashes?"
**Answer Focus**:
* **Kafka**:
  1. The crashed consumer stops sending heartbeats to the Group Coordinator broker.
  2. Once `session.timeout.ms` expires, the coordinator flags the consumer dead and triggers a **Consumer Group Rebalance**.
  3. Under the Eager protocol, all consumers in the group stop processing (Stop-The-World). Under the Cooperative Sticky protocol, only the partitions assigned to the dead consumer are reassigned.
  4. The newly assigned consumer queries `__consumer_offsets` for the last committed offset and resumes reading from that point. Any records processed between the last commit and the crash are **reprocessed (duplicates)**.
* **Pub/Sub**:
  1. No global rebalance occurs. There are no static partition assignments to redistribute.
  2. The messages currently leased by the crashed subscriber hit their **Ack Deadline**.
  3. The Pub/Sub Router detects the expiration (or missed lease extension heartbeat) and immediately returns those messages to the subscription's unacknowledged pool.
  4. Any other running subscriber pulling from that subscription receives those messages instantly. Unaffected subscribers experience zero pause or latency disruption.

---

### Q3: "Can you scale Kafka consumer instances beyond the number of topic partitions? How does this compare to Pub/Sub?"
**Answer Focus**:
* **In Kafka**:
  * **No (within the same consumer group)**. By design, a single partition can only be consumed by at most one consumer instance in a group at any given time to preserve ordering. Extra consumers will sit idle.
  * *Workaround*: A consumer can fetch batches and delegate record execution to internal worker thread pools, but this breaks partition-level ordering unless records with the same key are explicitly routed to the same worker thread.
* **In Pub/Sub**:
  * **Yes, completely unbounded**. Concurrency is completely decoupled from any partition count. You can have 500 subscriber instances pulling from a single subscription regardless of message volume, achieving dynamic work-stealing.

---

### Q4: "How does message ordering work when failures occur in Kafka vs Google Cloud Pub/Sub?"
**Answer Focus**:
* **In Kafka**:
  * Ordering is strictly guaranteed per partition. If record with offset 42 fails to process and the consumer halts its offset progress, offsets 43, 44, etc., are never consumed. The consumer either retries or sends record 42 to a Dead Letter Queue (DLQ) before advancing.
* **In Pub/Sub**:
  * Without ordering keys: Pub/Sub provides no ordering guarantees; redelivered messages arrive whenever available.
  * With `ordering_key`: When a message with an ordering key is NACKed or hits its Ack Deadline, **Pub/Sub halts delivery of all subsequent messages with that exact same ordering key** until the failing message is ACKed or successfully pushed to a Dead Letter Topic. Messages with different ordering keys continue processing unimpeded.

---

### Q5: "Explain the removal of ZooKeeper in Kafka (KRaft mode). What architectural problems did it solve?"
**Answer Focus**:
1. **Metadata Bottleneck**: In ZooKeeper mode, the single active controller broker had to synchronize partition states across ZooKeeper trees. Large clusters with $>200,000$ partitions experienced severe metadata lag.
2. **Controller Failover Latency**: When a controller crashed, the newly elected controller had to load the entire cluster state from ZooKeeper into memory, causing cluster freeze times ranging from 30 seconds to several minutes.
3. **Double Metadata Storage**: State was duplicated in both ZooKeeper and the controller's memory, leading to state desynchronization bugs.
4. **KRaft Solution**: Employs an event-driven Raft consensus log (`@metadata` topic). Controller metadata changes are committed as log records. Standby controllers have the metadata hot-replicated in memory. Controller failover occurs in **sub-seconds**, scaling Kafka safely to millions of partitions.

---

### Q6: "How do backpressure mechanisms differ between Kafka and Pub/Sub?"
**Answer Focus**:
* **Kafka**: Naturally **pull-driven backpressure**. Consumers invoke `poll(Duration)`. If downstream processing slows down (e.g., database latency), the consumer simply does not call `poll()` again or calls it with delay. The data stays safely buffered in the broker partition logs on disk.
  * *Gotcha*: If a consumer pauses processing longer than `max.poll.interval.ms`, Kafka assumes the consumer has hung and kicks it out of the consumer group.
* **Pub/Sub**:
  * **StreamingPull**: Managed via client SDK **Flow Control** settings (`FlowControlSettings`). You set limits on maximum outstanding messages (`setMaxOutstandingElementCount`) and bytes. The SDK pauses pulling from the gRPC stream when thresholds are reached.
  * **Push**: Backpressure relies on HTTP status codes. If the subscriber's endpoint returns HTTP 429 (Too Many Requests) or 503, Pub/Sub backs off dynamically using exponential backoff before retrying.

---

## 8. Common Pitfalls & Production Gotchas

### Apache Kafka Gotchas
> [!WARNING]
> **Rebalance Storms via `max.poll.interval.ms`**:
> If a batch of 500 records takes longer to process than `max.poll.interval.ms` (default 5 min), the coordinator marks the consumer dead and triggers a rebalance. Another consumer picks up the exact same batch, times out again, and causes an infinite cascading rebalance loop.
> *Fix*: Decrease `max.poll.records` or increase `max.poll.interval.ms`.

> [!CAUTION]
> **Partition Key Hot-Spotting**:
> If your partition key is poorly distributed (e.g., `tenant_id` where one enterprise tenant generates 90% of traffic), all their events hit a single partition, overloading one broker's disk/CPU while others sit idle.
> *Fix*: Use compound keys (`tenantId_userId`) or salted keys.

> [!IMPORTANT]
> **Uncommitted Offsets with Async Commit**:
> Calling `consumer.commitAsync()` without proper error handling or sequencing can result in a newer failed commit overwriting a successful older commit with a stale offset, causing unexpected reprocessing on crash.

---

### Google Cloud Pub/Sub Gotchas
> [!WARNING]
> **Duplicate Processing due to Slow Tasks exceeding Ack Deadline**:
> If your worker processes a task for 65 seconds, but the subscription `ackDeadlineSeconds` is 60 seconds (and lease management is unconfigured), Pub/Sub redelivers the message to *another* worker. Both workers now run the same business logic concurrently.
> *Fix*: Ensure `maxAckExtensionPeriod` is adequately configured or process idempotently.

> [!CAUTION]
> **Ordering Key Stalls on Unhandled Poison Pills**:
> When message ordering is enabled, an unhandled exception that continuously NACKs a message will pause processing for **all future messages with that ordering key indefinitely**.
> *Fix*: Configure a **Dead Letter Topic (DLT)** with a defined `maxDeliveryAttempts` to unblock the ordering pipeline automatically.

> [!NOTE]
> **High Cost on High-Volume Low-Latency Operations**:
> Pub/Sub charges per volume of data ingested ($40 per TiB). At multi-terabyte/petabyte continuous streaming scales, running dedicated self-managed Kafka clusters (or AWS MSK / Confluent) on reserved VM instances is frequently significantly cheaper than serverless Pub/Sub ingress fees.

---

## 9. Summary Cheat Sheet for Architecture & Interviews

```
+-----------------------------------------------------------------------------------------------+
| CRITERIA               | APACHE KAFKA                       | GOOGLE CLOUD PUB/SUB            |
+------------------------+------------------------------------+---------------------------------+
| Architectural Type     | Append-only distributed log        | Dynamic message routing broker  |
| Concurrency Unit       | Partition (Rigid)                  | Subscription (Elastic)          |
| Scaling Mechanism      | Add partitions & rebalance         | Auto-scales subscribers         |
| Ordering Scope         | Partition key                      | Ordering key                    |
| Offset Tracking        | Client tracks (offset pointer)     | Server tracks (per-message ACK) |
| Message Replay         | Native (rewind offset anytime)     | Seek snapshot / timestamp       |
| Primary Transport      | Custom TCP binary protocol         | gRPC / HTTPS                    |
| Stream Processing      | Kafka Streams, Flink, Spark        | Cloud Dataflow (Apache Beam)    |
| Operational Effort     | High (Broker disks, JVM, KRaft)    | Zero (Fully managed serverless) |
+-----------------------------------------------------------------------------------------------+
```
