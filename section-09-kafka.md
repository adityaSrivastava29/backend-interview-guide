---
layout: default
title: Kafka
nav_order: 10
---

# Section 9: Apache Kafka — Complete Guide

> Architecture, producers, consumers, Spring Kafka, and production patterns

---

## Table of Contents

1. [Kafka Architecture](#1-kafka-architecture)
2. [Topics, Partitions & Offsets](#2-topics-partitions--offsets)
3. [Producers](#3-producers)
4. [Consumers & Consumer Groups](#4-consumers--consumer-groups)
5. [Spring Kafka — Producer](#5-spring-kafka--producer)
6. [Spring Kafka — Consumer](#6-spring-kafka--consumer)
7. [Delivery Guarantees](#7-delivery-guarantees)
8. [Consumer Rebalancing](#8-consumer-rebalancing)
9. [Kafka Streams](#9-kafka-streams)
10. [Production Patterns & Problems](#10-production-patterns--problems)
11. [Interview Questions](#11-interview-questions)

---

## 1. Kafka Architecture

### Core Components

```
┌─────────────────────────────────────────────────────────────┐
│                        KAFKA CLUSTER                         │
│                                                              │
│  ┌──────────────────┐  ┌──────────────────┐                 │
│  │    BROKER 1      │  │    BROKER 2      │                 │
│  │ ┌──────────────┐ │  │ ┌──────────────┐ │                 │
│  │ │ orders-topic │ │  │ │ orders-topic │ │                 │
│  │ │ Partition 0  │ │  │ │ Partition 1  │ │  (replicated)   │
│  │ │ [LEADER]     │ │  │ │ [LEADER]     │ │                 │
│  │ └──────────────┘ │  │ └──────────────┘ │                 │
│  │ ┌──────────────┐ │  │ ┌──────────────┐ │                 │
│  │ │ Partition 1  │ │  │ │ Partition 0  │ │                 │
│  │ │ [FOLLOWER]   │ │  │ │ [FOLLOWER]   │ │                 │
│  │ └──────────────┘ │  │ └──────────────┘ │                 │
│  └──────────────────┘  └──────────────────┘                 │
│                                                              │
│  ┌──────────────────────────────────────────┐               │
│  │         ZOOKEEPER (or KRaft)             │               │
│  │  Broker coordination, leader election    │               │
│  └──────────────────────────────────────────┘               │
└─────────────────────────────────────────────────────────────┘
         ↑ Produce                   ↓ Consume
┌─────────────────┐         ┌─────────────────────┐
│    Producers    │         │    Consumer Groups  │
│ Order Service   │         │ Inventory Service   │
│ Payment Service │         │ Notification Service│
└─────────────────┘         └─────────────────────┘
```

### Why Kafka?

| Traditional Message Queue (RabbitMQ) | Kafka                                         |
| ------------------------------------ | --------------------------------------------- |
| Message deleted after consumption    | Message retained (configurable, e.g., 7 days) |
| Push-based                           | Pull-based (consumer controls rate)           |
| 1 consumer per message (competing)   | Multiple consumer groups read independently   |
| No replay                            | Replay from any offset                        |
| Limited throughput                   | Millions of messages/sec                      |
| No ordering                          | Ordering within partition                     |
| No stream processing                 | Kafka Streams built-in                        |

---

## 2. Topics, Partitions & Offsets

### Topic

A **Topic** is a logical stream of records. Think of it like a database table or a log file.

### Partition

A topic is split into **Partitions** — each partition is an ordered, immutable sequence of records.

```
orders-topic
├── Partition 0: [0] {order1} [1] {order5} [2] {order9} →
├── Partition 1: [0] {order2} [1] {order6} [2] {order10} →
├── Partition 2: [0] {order3} [1] {order7} [2] {order11} →
└── Partition 3: [0] {order4} [1] {order8} [2] {order12} →
```

### Offset

An **Offset** is the unique sequential position of a record within a partition.

```
Partition 0:
Offset:  0        1        2        3        4
Record: [msg_A] [msg_B] [msg_C] [msg_D] [msg_E] →

Consumer has read up to offset 2 → committed offset = 3 (next to read)
```

### Partitioning Strategy

```java
// Message WITHOUT key: round-robin across partitions
producer.send(new ProducerRecord<>("orders", null, orderJson));

// Message WITH key: same key → same partition (preserves order)
producer.send(new ProducerRecord<>("orders", orderId, orderJson));
// All events for orderId "ORD-123" → always same partition → ordered processing
```

### How Many Partitions?

```
Throughput per partition:
  Write: ~10 MB/s (depends on message size)
  Read: ~50 MB/s per consumer thread

Formula: max(target_throughput / partition_throughput, num_consumers)

Example:
  Target: 500 MB/s write
  Per partition: 10 MB/s
  Min partitions: 500/10 = 50 partitions

Rule of thumb: Start with 2x-3x expected consumer count
Avoid over-partitioning: each partition has overhead (file handles, memory)
```

---

## 3. Producers

### Producer Internals

```
Producer.send(record)
    ↓
Serializer (key + value → bytes)
    ↓
Partitioner (which partition?)
    ↓
RecordAccumulator (batch records by partition)
    ↓
Sender thread
    ↓
Network I/O (send batch to broker)
    ↓
Leader broker writes to partition log
    ↓
Followers replicate (if acks=all)
    ↓
Acknowledgment to producer
```

### Producer Configuration

```java
Map<String, Object> producerConfig = Map.of(
    ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka1:9092,kafka2:9092,kafka3:9092",
    ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class,
    ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class,

    // Acknowledgment policy
    ProducerConfig.ACKS_CONFIG, "all",         // wait for all replicas to ack

    // Retries
    ProducerConfig.RETRIES_CONFIG, 3,
    ProducerConfig.RETRY_BACKOFF_MS_CONFIG, 100,
    ProducerConfig.DELIVERY_TIMEOUT_MS_CONFIG, 120000,

    // Batching (improves throughput)
    ProducerConfig.BATCH_SIZE_CONFIG, 32768,     // 32KB batch
    ProducerConfig.LINGER_MS_CONFIG, 10,         // wait up to 10ms to accumulate batch
    ProducerConfig.COMPRESSION_TYPE_CONFIG, "snappy",  // compress batches

    // Idempotent producer (prevents duplicates from retries)
    ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true,

    // Buffer size
    ProducerConfig.BUFFER_MEMORY_CONFIG, 33554432L  // 32MB buffer
);
```

### Producer Acknowledgment (acks)

| acks         | Meaning                    | Durability                                      | Performance |
| ------------ | -------------------------- | ----------------------------------------------- | ----------- |
| `0`          | No ack — fire and forget   | Lowest (data loss possible)                     | Highest     |
| `1`          | Leader wrote               | Medium (lost if leader dies before replication) | Medium      |
| `all` / `-1` | All in-sync replicas wrote | Highest (no data loss)                          | Lowest      |

---

## 4. Consumers & Consumer Groups

### Consumer Group

Multiple consumers in the **same group** divide partitions among themselves — each partition assigned to exactly one consumer in the group.

```
orders-topic (4 partitions)
Consumer Group: "inventory-service"

  Consumer A (inventory-svc-pod-1): Partition 0, Partition 1
  Consumer B (inventory-svc-pod-2): Partition 2, Partition 3
  Consumer C (inventory-svc-pod-3): IDLE (more consumers than partitions!)

Consumer Group: "notification-service" (reads independently!)
  Consumer X (notif-svc-pod-1): Partition 0, Partition 1, Partition 2, Partition 3
```

Key insight: **Two consumer groups read the same topic independently** — each group has its own offset tracking.

### Offset Management

```
AUTO COMMIT: Consumer commits offsets automatically every auto.commit.interval.ms
  Risk: records processed but crashed before auto-commit → re-read (duplicate processing)

MANUAL COMMIT: Application commits after successful processing
  Safer: commit only after you're done with the message
```

---

## 5. Spring Kafka — Producer

### Setup

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

```yaml
spring:
  kafka:
    bootstrap-servers: kafka1:9092,kafka2:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all
      retries: 3
      batch-size: 32768
      linger-ms: 10
      compression-type: snappy
      properties:
        enable.idempotence: true
        max.in.flight.requests.per.connection: 5 # with idempotence, max 5
```

### Producer Code

```java
@Service
public class OrderEventPublisher {

    @Autowired
    private KafkaTemplate<String, OrderEvent> kafkaTemplate;

    // Simple send
    public void publishOrderPlaced(Order order) {
        OrderEvent event = OrderEvent.builder()
            .eventId(UUID.randomUUID().toString())
            .eventType("ORDER_PLACED")
            .orderId(order.getId())
            .userId(order.getUserId())
            .totalAmount(order.getTotalAmount())
            .occurredAt(Instant.now())
            .build();

        // Send with order ID as key → same order events go to same partition
        kafkaTemplate.send("order-events", order.getId().toString(), event);
    }

    // Send with callback (async)
    public void publishWithCallback(Order order) {
        OrderEvent event = buildEvent(order);

        kafkaTemplate.send("order-events", order.getId().toString(), event)
            .whenComplete((result, ex) -> {
                if (ex == null) {
                    log.info("Published to partition {} offset {}",
                        result.getRecordMetadata().partition(),
                        result.getRecordMetadata().offset());
                } else {
                    log.error("Failed to publish: {}", ex.getMessage());
                    // Alert, retry, or store in dead-letter queue
                }
            });
    }

    // Send and wait for acknowledgment (synchronous — blocks caller thread)
    public void publishSync(Order order) throws ExecutionException, InterruptedException {
        OrderEvent event = buildEvent(order);
        RecordMetadata metadata = kafkaTemplate.send("order-events",
            order.getId().toString(), event).get(); // blocks!
        log.info("Confirmed: partition={}, offset={}",
            metadata.partition(), metadata.offset());
    }

    // Transactional producer (all or nothing)
    @Transactional("kafkaTransactionManager")
    public void publishTransactionally(List<Order> orders) {
        orders.forEach(order ->
            kafkaTemplate.send("order-events", order.getId().toString(), buildEvent(order)));
        // All messages committed atomically, or none
    }
}
```

---

## 6. Spring Kafka — Consumer

### Configuration

```yaml
spring:
  kafka:
    consumer:
      group-id: inventory-service
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      auto-offset-reset: earliest # start from beginning if no committed offset
      enable-auto-commit: false # manual commit for reliability
      max-poll-records: 500 # max records per poll
      properties:
        spring.json.trusted.packages: "com.example.events"
    listener:
      ack-mode: MANUAL_IMMEDIATE # commit after each record
      concurrency: 3 # 3 threads = up to 3 partitions per instance
      type: BATCH # or SINGLE for one record at a time
```

### Consumer Code

```java
@Component
public class OrderEventConsumer {

    // Single message consumer
    @KafkaListener(
        topics = "order-events",
        groupId = "inventory-service",
        containerFactory = "kafkaListenerContainerFactory"
    )
    public void consumeOrderEvent(
            @Payload OrderEvent event,
            @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
            @Header(KafkaHeaders.OFFSET) long offset,
            Acknowledgment ack) {

        try {
            log.info("Processing event: {} from partition={} offset={}",
                event.getEventId(), partition, offset);

            inventoryService.reserveStock(event.getOrderId(), event.getItems());

            ack.acknowledge(); // commit offset ONLY after successful processing

        } catch (RetryableException e) {
            // Don't commit → message will be redelivered
            log.warn("Retryable error, will retry: {}", e.getMessage());
            throw e;
        } catch (NonRetryableException e) {
            // Business exception → send to DLT, then commit to not block queue
            deadLetterPublisher.send("order-events.DLT", event);
            ack.acknowledge(); // commit to skip the poison pill message
        }
    }

    // Batch consumer — more efficient
    @KafkaListener(topics = "order-events", groupId = "batch-inventory-service")
    public void consumeBatch(
            List<OrderEvent> events,
            Acknowledgment ack) {

        log.info("Processing batch of {} events", events.size());

        try {
            inventoryService.reserveStockBatch(events);
            ack.acknowledge();
        } catch (Exception e) {
            log.error("Batch processing failed: {}", e.getMessage());
            // Handle partial failures...
        }
    }

    // Multiple topics
    @KafkaListener(topics = {"order-events", "payment-events"})
    public void consumeMultipleTopics(ConsumerRecord<String, String> record) {
        String topic = record.topic();
        // route based on topic
    }
}
```

### Dead Letter Topic (DLT)

```java
@Configuration
public class KafkaErrorHandlingConfig {

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, OrderEvent>
            kafkaListenerContainerFactory(
                ConsumerFactory<String, OrderEvent> cf,
                KafkaTemplate<String, OrderEvent> template) {

        var factory = new ConcurrentKafkaListenerContainerFactory<String, OrderEvent>();
        factory.setConsumerFactory(cf);

        // Dead Letter configuration
        DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(
            template,
            // Route to {topic}.DLT
            (record, ex) -> new TopicPartition(record.topic() + ".DLT", record.partition())
        );

        // Retry 3 times with 1s backoff, then send to DLT
        DefaultErrorHandler errorHandler = new DefaultErrorHandler(
            recoverer,
            new FixedBackOff(1000L, 3)
        );

        // Don't retry on these exceptions
        errorHandler.addNotRetryableExceptions(
            IllegalArgumentException.class,
            JsonParseException.class
        );

        factory.setCommonErrorHandler(errorHandler);
        factory.getContainerProperties().setAckMode(AckMode.MANUAL_IMMEDIATE);

        return factory;
    }
}

// DLT consumer — manual retry or alerting
@KafkaListener(topics = "order-events.DLT", groupId = "order-dlt-handler")
public void handleDeadLetter(
        ConsumerRecord<String, OrderEvent> record,
        @Header(KafkaHeaders.EXCEPTION_MESSAGE) String errorMessage) {

    log.error("DLT message received from partition {} offset {}: {}",
        record.partition(), record.offset(), errorMessage);

    // Alert operations team
    alertingService.sendAlert("DLT message: " + errorMessage);

    // Or store in DB for manual investigation
    dltEventRepository.save(DltEvent.from(record, errorMessage));
}
```

---

## 7. Delivery Guarantees

### Three Levels

| Guarantee         | Description               | Risk                 | When to Use                         |
| ----------------- | ------------------------- | -------------------- | ----------------------------------- |
| **At-most-once**  | Fire and forget, no retry | Data loss possible   | Metrics, logs (loss OK)             |
| **At-least-once** | Retry on failure          | Duplicate processing | Most use cases (handle idempotency) |
| **Exactly-once**  | No loss, no duplicate     | Complex, overhead    | Payments, financial                 |

### At-Least-Once (Default)

```java
// Producer: acks=all + retries + idempotent=true
// Consumer: manual commit AFTER processing

// Risk scenario:
1. Consumer processes message
2. Consumer crashes BEFORE committing offset
3. Consumer restarts, re-reads message, processes AGAIN
→ Duplicate! Handle with idempotency check.
```

### Exactly-Once (Transactional API)

```java
// Producer: enable.idempotence=true + transactional.id
// Consumer: isolation.level=read_committed

// Producer
@Bean
public ProducerFactory<String, OrderEvent> exactlyOnceProducerFactory() {
    Map<String, Object> config = new HashMap<>();
    config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
    config.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "order-service-txn-1");
    return new DefaultKafkaProducerFactory<>(config);
}

// Consume-Transform-Produce pattern (EOS)
@KafkaListener(topics = "raw-orders")
public void processExactlyOnce(OrderEvent event, Acknowledgment ack) {
    kafkaTemplate.executeInTransaction(operations -> {
        // All these happen atomically
        EnrichedOrder enriched = enrichOrder(event);
        operations.send("enriched-orders", enriched.getOrderId(), enriched);
        ack.acknowledge();
        return enriched;
    });
}
```

---

## 8. Consumer Rebalancing

### What is Rebalancing?

When consumers join or leave a group, Kafka **reassigns partitions** among available consumers.

```
Initial state: 4 partitions, 2 consumers
  Consumer A: P0, P1
  Consumer B: P2, P3

Consumer C joins:
  Consumer A: P0, P1 ← gets rebalanced
  Consumer B: P2     ← gets rebalanced
  Consumer C: P3

Consumer A dies:
  Consumer B: P0, P1, P2
  Consumer C: P3
```

### Problem: Rebalancing Causes Processing Gaps

```
During rebalance:
1. All consumers STOP processing (stop-the-world)
2. Coordinator assigns partitions
3. Consumers restart with new assignments
→ Latency spike, processing pause
```

### Solution: Cooperative Sticky Rebalancing

```yaml
spring:
  kafka:
    consumer:
      properties:
        partition.assignment.strategy: org.apache.kafka.clients.consumer.CooperativeStickyAssignor
        # Incremental rebalancing: only reassign necessary partitions
        # Consumers not affected can keep processing
```

### Preventing Rebalance During Processing

```yaml
spring:
  kafka:
    consumer:
      max-poll-interval-ms: 300000 # 5 min — max time between polls (must be > processing time)
      session-timeout-ms: 30000 # broker considers consumer dead if no heartbeat in 30s
      heartbeat-interval-ms: 10000 # heartbeat every 10s (must be < session-timeout)
      max-poll-records: 100 # fewer records → faster processing → fewer rebalance triggers
```

---

## 9. Kafka Streams

### Definition

Kafka Streams is a library for **stream processing** directly within Kafka — no separate processing cluster needed (unlike Spark/Flink).

```java
@Configuration
@EnableKafkaStreams
public class OrderStreamProcessor {

    @Bean
    public KStream<String, OrderEvent> orderEnrichmentStream(StreamsBuilder builder) {
        KStream<String, OrderEvent> orders = builder.stream("raw-orders",
            Consumed.with(Serdes.String(), new JsonSerde<>(OrderEvent.class)));

        // Filter: only process completed orders
        KStream<String, OrderEvent> completedOrders = orders
            .filter((key, order) -> "COMPLETED".equals(order.getStatus()));

        // Enrich: join with user data from a table (KTable backed by "users" topic)
        KTable<String, User> users = builder.table("users",
            Consumed.with(Serdes.String(), new JsonSerde<>(User.class)));

        KStream<String, EnrichedOrder> enriched = completedOrders
            .join(users,
                (order, user) -> new EnrichedOrder(order, user),
                Joined.with(Serdes.String(), new JsonSerde<>(OrderEvent.class), new JsonSerde<>(User.class)));

        // Aggregate: count orders per user in 1-hour windows
        KTable<Windowed<String>, Long> orderCounts = orders
            .groupByKey()
            .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofHours(1)))
            .count();

        // Write to output topic
        enriched.to("enriched-orders",
            Produced.with(Serdes.String(), new JsonSerde<>(EnrichedOrder.class)));

        return enriched;
    }
}
```

---

## 10. Production Patterns & Problems

### Pattern 1: Idempotent Consumer

```java
@Component
public class IdempotentOrderConsumer {

    @Autowired
    private ProcessedEventRepository processedEventRepo;

    @KafkaListener(topics = "order-events")
    @Transactional
    public void process(OrderEvent event, Acknowledgment ack) {

        // Idempotency check: use eventId as unique key
        if (processedEventRepo.existsByEventId(event.getEventId())) {
            log.info("Duplicate event skipped: {}", event.getEventId());
            ack.acknowledge();
            return;
        }

        try {
            inventoryService.reserve(event);
            processedEventRepo.save(new ProcessedEvent(event.getEventId(), Instant.now()));
            ack.acknowledge();
        } catch (Exception e) {
            // Don't ack — will be redelivered
            throw e;
        }
    }
}
```

### Pattern 2: Message Ordering Guarantee

```java
// Ensure all events for an order go to same partition → guaranteed order
producer.send(new ProducerRecord<>(
    "order-events",
    order.getId().toString(),  // KEY → determines partition
    event
));

// Consumer: partitions are consumed sequentially within a consumer thread
// → Events for same orderId arrive in order
```

### Problem: Consumer Lag

**Symptom:** Kafka consumer lag growing (messages piling up faster than consumed).

**Diagnosis:**

```bash
# Check consumer group lag
kafka-consumer-groups.sh --bootstrap-server kafka:9092 \
  --group inventory-service --describe

# Output:
# TOPIC          PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
# order-events   0          5000            15000           10000  ← 10k lag!
```

**Solutions:**

1. **Increase consumer instances** (scale deployment, max = num partitions)
2. **Batch processing** — increase `max.poll.records`
3. **Parallel processing within consumer**:

```java
@KafkaListener(topics = "order-events", concurrency = "6") // 6 consumer threads
public void process(OrderEvent event, Acknowledgment ack) { ... }
```

4. **Speed up processing** — optimize DB queries, async processing

### Problem: Poison Pill Message

**Symptom:** Consumer stuck, same error repeating, consumer lag growing.

**Cause:** Message with invalid payload that can't be deserialized or processed.

**Solution:**

```java
// Deserializer that doesn't throw on error
props.put(ErrorHandlingDeserializer.VALUE_DESERIALIZER_CLASS, JsonDeserializer.class);
props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, ErrorHandlingDeserializer.class);

// Error handler sends poison pill to DLT
DefaultErrorHandler errorHandler = new DefaultErrorHandler(
    new DeadLetterPublishingRecoverer(kafkaTemplate),
    new FixedBackOff(1000L, 2L) // 2 retries, then DLT
);
errorHandler.addNotRetryableExceptions(DeserializationException.class);
```

---

## 11. Interview Questions

#### Basic

1. What is Apache Kafka and what problems does it solve?
2. What is the difference between a topic and a partition?
3. What is a consumer group?

#### Intermediate

4. Explain the three delivery guarantees in Kafka.
5. How does Kafka ensure message ordering?
6. What is consumer lag and how do you reduce it?

#### Advanced

7. Explain the at-least-once vs exactly-once semantics implementation in Kafka.
8. How does idempotent producer work internally?
9. What is Consumer Rebalancing and how does Cooperative Sticky strategy improve it?
10. How would you design a Kafka-based order processing pipeline with exactly-once semantics?

#### Scenario-Based

11. _Your Kafka consumer group has 10 consumers but only 6 partitions. What happens?_

    - 6 consumers active (one per partition), 4 consumers IDLE. More consumers than partitions is wasteful.

12. _How would you handle a "poison pill" message that causes the consumer to crash on every read?_

    - Retry N times → Dead Letter Topic → manual investigation → fix and replay

13. _Order service publishes order-placed, inventory reserves stock, but inventory consumer crashes mid-processing. How do you prevent duplicate reservations on restart?_
    - Idempotent consumer with `event_id` unique constraint check

---

### Summary — Kafka Cheatsheet

```
Core Concepts:
  Topic: logical stream (like a table)
  Partition: ordered log within topic (parallel unit)
  Offset: position within partition
  Consumer Group: consumers sharing partitions

Key Rule: num_consumers ≤ num_partitions for full parallelism

Producer acks:
  acks=0: fire and forget (data loss possible)
  acks=1: leader acked (replica lag risk)
  acks=all: all ISR acked (safest)

Enable idempotence: prevents duplicates from producer retries

Consumer:
  auto-commit: risky (commit before process complete)
  manual commit (MANUAL_IMMEDIATE): commit after success

Delivery Guarantees:
  at-most-once: no retry
  at-least-once: retry + idempotent consumer
  exactly-once: transactional API + read_committed

Rebalancing: partitions reassigned when consumers join/leave
  → Use CooperativeStickyAssignor for incremental rebalancing
  → Tune max-poll-interval-ms > processing time to avoid false rebalance

Dead Letter Topic: poison pills → DLT for investigation
Consumer Lag: diagnose with kafka-consumer-groups → scale consumers or batch
Ordering: same key → same partition → ordered processing
```
