---
layout: default
title: Production Support
nav_order: 15
---

# Section 14: Production Support — Incidents, Debugging & Performance

> Real-world production incidents, root cause analysis, and how to fix them

---

## Table of Contents

1. [Debugging Methodology](#1-debugging-methodology)
2. [JVM & Memory Issues](#2-jvm--memory-issues)
3. [Database Performance Issues](#3-database-performance-issues)
4. [Concurrency & Thread Issues](#4-concurrency--thread-issues)
5. [Spring Boot Runtime Issues](#5-spring-boot-runtime-issues)
6. [Kafka Consumer Issues](#6-kafka-consumer-issues)
7. [Distributed System Issues](#7-distributed-system-issues)
8. [Performance Tuning Guide](#8-performance-tuning-guide)
9. [Interview Questions](#9-interview-questions)

---

# 1. Debugging Methodology

## SRE Incident Response Playbook

```
1. DETECT: Alert fires or user reports issue
2. TRIAGE: What is the user impact? P1/P2/P3?
3. COMMUNICATE: Post in #incidents, notify stakeholders
4. INVESTIGATE:
   - Check dashboards: error rate, latency, traffic
   - Check recent deployments (most common cause!)
   - Read error logs
5. MITIGATE: Rollback, redirect traffic, scale up, feature flag off
6. ROOT CAUSE: Once stable, find WHY it happened
7. POSTMORTEM: Document timeline, root cause, action items
```

## The Most Valuable Production Debugging Commands

```bash
# 1. See recent deployments
kubectl rollout history deployment/order-service

# 2. Check pod logs with grep
kubectl logs -l app=order-service --tail=1000 | grep -i "error\|exception\|warn"

# 3. Get a thread dump (diagnose deadlocks, stuck threads)
kubectl exec -it order-service-xyz -- jstack -l 1

# 4. Get heap histogram (what's consuming memory?)
kubectl exec -it order-service-xyz -- jmap -histo:live 1 | head -50

# 5. Check GC activity
kubectl exec -it order-service-xyz -- jstat -gcutil 1 1000 30

# 6. Check active threads/connections via Actuator
curl http://service:8080/actuator/metrics/executor.active
curl http://service:8080/actuator/metrics/hikaricp.connections.active

# 7. Trigger heap dump for deep analysis
kubectl exec -it order-service-xyz -- jmap -dump:live,format=b,file=/tmp/heap.hprof 1
kubectl cp order-service-xyz:/tmp/heap.hprof ./heap.hprof
# Open in Eclipse MAT or VisualVM
```

---

# 2. JVM & Memory Issues

## Incident: OutOfMemoryError: Java Heap Space

### Symptoms
- `java.lang.OutOfMemoryError: Java heap space` in logs
- Pod crashes and restarts (Kubernetes OOMKilled)
- GC time > 95% (CPU spike before OOM)

### Investigation

```bash
# Check if OOMKilled
kubectl describe pod order-service-xyz | grep -A3 "OOMKilled\|Reason"

# Check GC activity
kubectl exec -it order-service-xyz -- jstat -gcutil 1 1000 60
# Output: S0  S1  E    O    M    YGC  YGCT  FGC  FGCT   GCT
#         0.00 80.00 100  99.9  95   2000  20.00  50  500   520
# O=99.9% (Old Gen almost full) FGC=50 (lots of full GCs) → imminent OOM

# Take heap dump before it dies
kubectl exec -it order-service-xyz -- jmap -dump:live,format=b,file=/tmp/heap.hprof 1

# Quick histogram
kubectl exec -it order-service-xyz -- jmap -histo:live 1 | head -30
# Look for: large counts, large bytes, unexpected classes
```

### Common Root Causes & Fixes

**1. Unbounded Cache**
```java
// BAD: Cache with no eviction — grows forever!
private Map<String, Product> productCache = new HashMap<>();

// GOOD: Bounded LRU cache
private Map<String, Product> productCache = new LinkedHashMap<String, Product>(1000, 0.75f, true) {
    protected boolean removeEldestEntry(Map.Entry eldest) {
        return size() > 1000;
    }
};

// BETTER: Spring Cache with TTL + max size
@Bean
public CacheManager cacheManager() {
    return new ConcurrentMapCacheManager(); // with Caffeine:
}
// Caffeine: maximumSize(10000).expireAfterWrite(1, HOURS)
```

**2. Session/Connection Leak**
```java
// BAD: InputStream not closed
InputStream is = url.openStream();
// ... process ... 
// forgot to close → resource leak → eventually OOM

// GOOD: try-with-resources
try (InputStream is = url.openStream()) {
    // ... process ...
} // auto-closed
```

**3. Static Collection Growth**
```java
// BAD: Static list grows forever with every request
public class MetricsCollector {
    private static final List<RequestMetric> metrics = new ArrayList<>(); // DANGER
    
    public void record(RequestMetric metric) {
        metrics.add(metric); // unbounded growth!
    }
}

// GOOD: Use rolling window or external metrics system
// Or cap the list size
```

## Incident: OutOfMemoryError: Metaspace

```bash
# Symptoms: Metaspace full, class loading failure
java.lang.OutOfMemoryError: Metaspace

# Root causes:
# 1. Too many classes loaded (frameworks, CGLIB proxies)
# 2. Class loader leak (common with hot reloading in production)

# Fix: Increase Metaspace limit
-XX:MaxMetaspaceSize=512m  # default is unlimited (grows until native memory OOM)

# Or find the leak:
-verbose:class 2>&1 | grep "Loaded" | wc -l  # count loaded classes over time
```

## Incident: High GC Pause (STW > 2 seconds)

```bash
# Enable GC logging
-Xlog:gc*:file=/tmp/gc.log:time,uptime,level,tags:filecount=5,filesize=20m

# Analyze with GCViewer or GCEasy.io
# Look for: Full GC frequency, pause duration, promotion failure

# Switch to ZGC for low-pause (< 1ms pause even with 1TB heap)
-XX:+UseZGC -XX:SoftMaxHeapSize=4g
```

---

# 3. Database Performance Issues

## Incident: Slow Query Causing API Timeout

### Symptoms
- API endpoints timing out with 504
- DB CPU at 100%
- Slow query log filling up

### Investigation

```sql
-- PostgreSQL: Find slow queries currently running
SELECT pid, now() - pg_stat_activity.query_start AS duration, query, state
FROM pg_stat_activity
WHERE state != 'idle'
  AND now() - pg_stat_activity.query_start > interval '5 seconds'
ORDER BY duration DESC;

-- Find most expensive queries (by total time)
SELECT query, calls, total_exec_time, mean_exec_time, rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Explain a slow query
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT o.*, u.name FROM orders o 
JOIN users u ON u.id = o.user_id
WHERE o.status = 'PENDING'
ORDER BY o.created_at DESC
LIMIT 20;

-- Look for:
-- Seq Scan on large table → missing index
-- Nested Loop with large outer → wrong join type, missing index
-- Buffers: read=huge → lots of I/O, needs better caching or index
```

### Fix: Add Missing Index

```sql
-- Missing index diagnosis
EXPLAIN SELECT * FROM orders WHERE user_id = 1001 AND status = 'PENDING';
-- Seq Scan on orders (cost=0..450000) → BAD, full table scan!

-- Add composite index
CREATE INDEX CONCURRENTLY idx_orders_user_status 
ON orders(user_id, status) 
WHERE status IN ('PENDING', 'PROCESSING'); -- partial index — only index relevant rows

EXPLAIN SELECT * FROM orders WHERE user_id = 1001 AND status = 'PENDING';
-- Index Scan using idx_orders_user_status → GOOD!
```

## Incident: Connection Pool Exhaustion

### Symptoms
```
HikariPool-1 - Connection is not available, request timed out after 30000ms
```

### Investigation & Fix

```bash
# Check active connections
curl http://service:8080/actuator/metrics/hikaricp.connections.active
curl http://service:8080/actuator/metrics/hikaricp.connections.pending

# Check where connections are held (thread dump to find long-running transactions)
jstack PID | grep -A 20 "BLOCKED\|WAITING"
```

```java
// Common cause 1: @Transactional method calling external API
@Transactional
public Order processOrder(OrderRequest request) {
    orderRepository.save(new Order(request));
    paymentGateway.charge(request.getPayment()); // SLOW: 3rd party API holds DB connection!
    return orderRepository.findById(orderId);
}

// Fix: Move external call outside transaction
public Order processOrder(OrderRequest request) {
    // Save order first, then release DB connection
    Long orderId = saveOrder(request);       // transaction 1 — fast
    paymentGateway.charge(request.getPayment()); // slow, but no DB connection held
    return confirmOrder(orderId);            // transaction 2 — fast
}

@Transactional
private Long saveOrder(OrderRequest request) { ... }

// Common cause 2: Pool too small for load
spring:
  datasource:
    hikari:
      maximum-pool-size: 20   # adjust based on DB max_connections / num_pods
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
```

## Incident: Deadlock

```
ERROR: could not serialize access due to concurrent update
ERROR: deadlock detected
```

```java
// Root cause: two transactions acquiring locks in opposite order
// Transaction A: Lock row 1, then row 2
// Transaction B: Lock row 2, then row 1
// → Deadlock!

// Fix 1: Always acquire locks in same order
@Transactional
public void transferMoney(Account from, Account to, BigDecimal amount) {
    // Always lock by ID order (lower ID first)
    List<Long> ids = List.of(from.getId(), to.getId()).stream().sorted().toList();
    
    // Lock both accounts in consistent order
    accountRepository.findByIdWithLock(ids.get(0));
    accountRepository.findByIdWithLock(ids.get(1));
    
    from.debit(amount);
    to.credit(amount);
}

// Fix 2: Short transactions (hold locks briefly)
// Fix 3: Retry on deadlock
@Retryable(retryFor = {DeadlockLoserDataAccessException.class}, maxAttempts = 3)
@Transactional
public void updateOrder(Order order) { ... }
```

---

# 4. Concurrency & Thread Issues

## Incident: Thread Pool Exhaustion (API Timeouts)

```java
// Symptom: All API calls timing out, thread pool full
// jstack shows all threads BLOCKED or WAITING on same lock/resource

// Root cause: slow operation in request thread
@PostMapping("/send-email")
public void sendEmail(@RequestBody EmailRequest request) {
    emailService.send(request);  // 2-3 seconds per email! Blocks request thread
}

// Fix: Async processing
@PostMapping("/send-email")
public void sendEmail(@RequestBody EmailRequest request) {
    emailService.sendAsync(request);  // returns immediately
}

@Service
public class EmailService {
    @Async("emailTaskExecutor")
    public CompletableFuture<Void> sendAsync(EmailRequest request) {
        // Runs in separate thread pool, doesn't block API threads
        smtpClient.send(request);
        return CompletableFuture.completedFuture(null);
    }
}

@Bean("emailTaskExecutor")
public ThreadPoolTaskExecutor emailTaskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(5);
    executor.setMaxPoolSize(20);
    executor.setQueueCapacity(100);
    executor.setThreadNamePrefix("email-");
    executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
    return executor;
}
```

## Incident: Race Condition — Overselling

```java
// Symptom: Inventory shows -5 units (sold more than in stock)
// Root cause: non-atomic check-and-update

// BAD: Non-atomic read-check-write
public void sellProduct(Long productId, int qty) {
    Product p = productRepo.findById(productId).get();
    if (p.getStock() >= qty) {           // Thread A reads stock=10, Thread B reads stock=10
        p.setStock(p.getStock() - qty);  // Thread A sets 8, Thread B sets 8 → not 6!
        productRepo.save(p);
    }
}

// FIX 1: Optimistic locking (@Version)
@Entity
public class Product {
    @Version
    private int version; // incremented on each save
}
// If concurrent update → OptimisticLockingFailureException → retry

// FIX 2: Pessimistic locking
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT p FROM Product p WHERE p.id = :id")
Product findByIdWithLock(@Param("id") Long id);

// FIX 3: Atomic SQL update
@Modifying
@Query("UPDATE Product p SET p.stock = p.stock - :qty WHERE p.id = :id AND p.stock >= :qty")
int decrementStock(@Param("id") Long id, @Param("qty") int qty);
// Returns 0 if insufficient stock (stock < qty), 1 if success
```

---

# 5. Spring Boot Runtime Issues

## Incident: @Transactional Not Working

### Case 1: Checked Exception Not Rolling Back

```java
// BAD: Checked exceptions don't trigger rollback by default!
@Transactional
public void processOrder(Order order) throws PaymentException {
    orderRepository.save(order);
    payment.charge(order);  // Throws PaymentException (checked)
    // ORDER SAVED, PAYMENT FAILED — inconsistent state!
}

// FIX: Include checked exceptions
@Transactional(rollbackFor = {PaymentException.class, Exception.class})
public void processOrder(Order order) throws PaymentException { ... }
```

### Case 2: Self-Invocation Bypass

```java
// BAD: Calling @Transactional method from same class
@Service
public class OrderService {
    
    public void placeOrder(Order order) {
        processPayment(order);  // DIRECT CALL — bypasses proxy → no transaction!
    }
    
    @Transactional
    public void processPayment(Order order) {
        // This runs WITHOUT a transaction!
    }
}

// FIX: Inject self or extract to separate bean
@Service
public class OrderService {
    @Autowired
    private OrderService self;  // inject proxy
    
    public void placeOrder(Order order) {
        self.processPayment(order);  // goes through proxy → transaction works
    }
    
    @Transactional
    public void processPayment(Order order) { ... }
}
```

### Case 3: @Async + @Transactional

```java
// BAD: @Async breaks transaction propagation
@Transactional
public void saveOrder(Order order) {
    orderRepo.save(order);
    notifyAsync(order);  // Async — runs in new thread, no transaction context
}

@Async
@Transactional  // NEW transaction in new thread — won't rollback with parent!
public void notifyAsync(Order order) {
    notificationRepo.save(new Notification(order));
}
```

## Incident: Bean Creation Failure (Circular Dependency)

```
The dependencies of some of the beans in the application context form a cycle:
OrderService → PaymentService → OrderService
```

```java
// FIX 1: Use @Lazy
@Service
public class OrderService {
    @Autowired
    @Lazy  // inject lazily — proxy injected, actual bean created on first use
    private PaymentService paymentService;
}

// FIX 2 (Better): Redesign to break the cycle
// Usually indicates a design problem — extract the shared dependency
@Service
public class OrderEventPublisher {  // extracted shared logic
    public void publishOrderEvent(Order order) { ... }
}

// Both OrderService and PaymentService inject OrderEventPublisher (no cycle)
```

---

# 6. Kafka Consumer Issues

## Incident: Consumer Stuck on Poison Pill

### Symptoms
- Consumer lag growing
- Same error in logs repeatedly: `Failed to deserialize`
- No progress on partition

### Fix

```java
// Configure error handler with DLT
@Bean
public DefaultErrorHandler errorHandler(KafkaTemplate<String, Object> template) {
    DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(template);
    DefaultErrorHandler handler = new DefaultErrorHandler(recoverer, 
        new FixedBackOff(1000L, 2L)); // retry 2x, then DLT
    handler.addNotRetryableExceptions(DeserializationException.class);
    return handler;
}
```

## Incident: Consumer Lag Growing (Processing Too Slow)

### Diagnosis

```bash
kafka-consumer-groups.sh --bootstrap-server kafka:9092 \
  --group order-processor --describe
```

### Solutions

```java
// Option 1: Increase concurrency (one thread per partition)
@KafkaListener(topics = "orders", concurrency = "6")
public void process(OrderEvent event) { ... }

// Option 2: Batch processing
@KafkaListener(topics = "orders", containerFactory = "batchFactory")
public void processBatch(List<OrderEvent> events) {
    // Process in batches — less DB round-trips
    orderService.createOrdersBatch(events);
}

// Option 3: Async within listener
@KafkaListener(topics = "orders")
public void process(OrderEvent event, Acknowledgment ack) {
    CompletableFuture.runAsync(() -> orderService.create(event), executor)
        .thenRun(ack::acknowledge)
        .exceptionally(ex -> {
            log.error("Async processing failed", ex);
            return null;
        });
}
```

---

# 7. Distributed System Issues

## Incident: Cascading Failure

```
Payment gateway slow → Order service threads exhausted → API gateway requests queue up
→ API gateway thread pool exhausted → all APIs down
```

### Solution Applied

```java
// 1. Timeout: don't wait indefinitely
@Bean
public OkHttpClient httpClient() {
    return new OkHttpClient.Builder()
        .connectTimeout(Duration.ofSeconds(2))
        .readTimeout(Duration.ofSeconds(5))
        .build();
}

// 2. Circuit Breaker: stop calling failing service
@CircuitBreaker(name = "paymentGateway", fallbackMethod = "paymentFallback")
@TimeLimiter(name = "paymentGateway")
public CompletableFuture<PaymentResult> charge(PaymentRequest req) { ... }

// 3. Bulkhead: isolate thread pool
@Bulkhead(name = "paymentGateway")
public PaymentResult charge(PaymentRequest req) { ... }

// 4. Fallback: graceful degradation
public CompletableFuture<PaymentResult> paymentFallback(PaymentRequest req, Exception e) {
    retryQueue.add(req);
    return CompletableFuture.completedFuture(PaymentResult.pending("Payment queued"));
}
```

## Incident: Duplicate Processing

```
Scenario: Kafka consumer processes an order event and saves to DB.
Consumer crashes before committing offset.
Restart → event reprocessed → duplicate order!
```

```java
// Fix: Idempotent consumer
@KafkaListener(topics = "order-events")
@Transactional
public void processOrderEvent(OrderEvent event, Acknowledgment ack) {
    
    // Atomic upsert using unique constraint on event_id
    try {
        ProcessedEvent processed = new ProcessedEvent(event.getEventId());
        processedEventRepo.save(processed); // will throw if duplicate (UNIQUE constraint)
        
        orderService.createOrder(event);
        ack.acknowledge();
        
    } catch (DataIntegrityViolationException e) {
        // Duplicate event — already processed
        log.info("Duplicate event ignored: {}", event.getEventId());
        ack.acknowledge();
    }
}
```

---

# 8. Performance Tuning Guide

## JVM Tuning

```bash
# Production JVM flags for Java 17+ Spring Boot
java \
  -XX:+UseContainerSupport \
  -XX:MaxRAMPercentage=75.0 \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=200 \
  -XX:+ExplicitGCInvokesConcurrent \
  -XX:+UseStringDeduplication \
  -Xss512k \               # stack size per thread (reduce if many threads)
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/tmp/heap-$(date +%s).hprof \
  -Xlog:gc*:file=/tmp/gc.log:time,uptime:filecount=5,filesize=20m \
  -jar app.jar

# For ZGC (Java 15+) — sub-millisecond pauses
  -XX:+UseZGC \
  -XX:SoftMaxHeapSize=3g    # soft cap, GC kicks in before hard limit
```

## Database Query Optimization Checklist

```
1. EXPLAIN ANALYZE every query before deploying
2. Add indexes for:
   - WHERE clause columns
   - JOIN columns  
   - ORDER BY columns (if used with LIMIT)
   - Composite: most selective column first
3. Avoid:
   - SELECT * (fetch only needed columns)
   - N+1 queries (use JOIN FETCH or @EntityGraph)
   - OFFSET for pagination (use keyset/cursor instead)
   - Functions on indexed columns in WHERE (loses index)
4. Connection pooling:
   - Hikari max-pool-size ≈ DB max_connections / num_pods
   - Keep transactions short
   - Never call external APIs inside @Transactional
5. Read replicas:
   - Route read-only queries to replicas
   - Use @Transactional(readOnly = true) for queries
```

## API Latency Optimization

```java
// 1. Parallel service calls instead of sequential
// BAD: sequential (total = 200ms + 300ms = 500ms)
ProductDetails product = productService.get(id);   // 200ms
InventoryStatus stock = inventoryService.get(id);  // 300ms

// GOOD: parallel (total = max(200ms, 300ms) = 300ms)
CompletableFuture<ProductDetails> productFuture = 
    CompletableFuture.supplyAsync(() -> productService.get(id));
CompletableFuture<InventoryStatus> stockFuture = 
    CompletableFuture.supplyAsync(() -> inventoryService.get(id));
CompletableFuture.allOf(productFuture, stockFuture).join();

// 2. Caching (aim for > 90% cache hit rate)
// 3. Async for non-critical path (notifications, audit logs)
// 4. Database query optimization
// 5. Pagination (never return unbounded results)
```

---

# 9. Interview Questions

### Debugging
1. Walk me through how you'd debug a memory leak in production.
2. How do you identify a slow database query?
3. What is a thread dump and when do you use it?

### Incidents
4. How would you debug `HikariPool - Connection is not available`?
5. You deployed and error rate went from 0.1% to 15%. What do you do?
6. A Kafka consumer's lag is growing. Walk through your diagnosis.

### Production Readiness
7. What metrics would you instrument in a Java microservice?
8. How do you ensure zero-downtime deployments with DB schema changes?
9. How do you handle the "poison pill" problem in Kafka?

---

## Summary — Production Support Cheatsheet

```
First Response:
  1. Check dashboards: error rate, latency, traffic drop
  2. Check recent deployments → git blame the incident
  3. Read error logs: grep -i "error\|exception"
  4. Rollback if deployment caused it

JVM Issues:
  OOM: heap dump → Eclipse MAT → find large object trees
  High GC: jstat -gcutil → GCEasy.io → tune heap / reduce allocations
  Thread issues: jstack → find BLOCKED threads → find root lock

DB Issues:
  Slow query: pg_stat_statements → EXPLAIN ANALYZE → add index
  Deadlock: consistent lock ordering + short transactions + retry
  Pool exhaustion: move external calls outside @Transactional

Spring Issues:
  @Transactional not working: self-invocation / checked exception / @Async
  Circular dependency: @Lazy or redesign

Kafka Issues:
  Consumer lag: increase concurrency / batch processing
  Poison pill: DLT + error handler + max retries

Distributed Issues:
  Cascading failure: timeout + circuit breaker + bulkhead
  Duplicate processing: idempotency check on event ID
```
