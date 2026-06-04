# Section 15: Interview Q&A Bank — Top Questions with Answers

> Curated top questions across Java, Spring Boot, Microservices, Kafka, and SQL with concise answers

---

## Table of Contents

1. [Core Java — Top 50 Questions](#1-core-java--top-50-questions)
2. [Spring Boot & Spring Framework — Top 40 Questions](#2-spring-boot--spring-framework--top-40-questions)
3. [Microservices & Distributed Systems — Top 30 Questions](#3-microservices--distributed-systems--top-30-questions)
4. [Apache Kafka — Top 25 Questions](#4-apache-kafka--top-25-questions)
5. [Databases & SQL — Top 25 Questions](#5-databases--sql--top-25-questions)
6. [Redis — Top 20 Questions](#6-redis--top-20-questions)
7. [System Design — Top 15 Questions](#7-system-design--top-15-questions)
8. [Behavioral & Situational — Top 15 Questions](#8-behavioral--situational--top-15-questions)

---

# 1. Core Java — Top 50 Questions

---

**Q1. Explain JVM architecture.**

> The JVM has three main parts: **Class Loader** (loads .class files), **Memory Areas** (Heap, Stack, Metaspace, Code Cache), and **Execution Engine** (interpreter + JIT compiler). The JIT compiles hot bytecode to native machine code after profiling.

---

**Q2. What is the difference between Heap and Stack memory?**

| Heap | Stack |
|------|-------|
| Objects and arrays | Method frames, local variables, references |
| Shared across all threads | Per-thread |
| GC managed | Automatically reclaimed when method returns |
| Slow (GC overhead) | Fast (LIFO) |
| `OutOfMemoryError` when full | `StackOverflowError` when full |

---

**Q3. Explain HashMap internal working.**

> HashMap uses an array of `Node<K,V>` buckets. On `put(key, value)`: compute `hash(key)`, find bucket index (`hash & (n-1)`), insert as a linked list node. If bucket has ≥ 8 nodes and capacity ≥ 64, the linked list is converted to a **Red-Black Tree** (O(log n) lookup). On `get(key)`: compute hash, navigate to bucket, compare with `equals()`.

---

**Q4. What is the hashCode() and equals() contract?**

> - If `a.equals(b)` is true, then `a.hashCode()` must equal `b.hashCode()`
> - If `a.hashCode() == b.hashCode()`, `a.equals(b)` may be false (hash collision is OK)
> - Violation: same-content objects could land in different HashMap buckets → **key lookup returns null** even though key exists!

---

**Q5. What is the difference between ArrayList and LinkedList?**

| Operation | ArrayList | LinkedList |
|-----------|-----------|------------|
| Random access `get(i)` | O(1) | O(n) |
| Add to end | O(1) amortized | O(1) |
| Add/remove from middle | O(n) (shift) | O(1) (pointer update) |
| Memory | Compact (contiguous array) | High (each node has prev+next pointers) |

> **Use ArrayList by default.** LinkedList better only when frequent insertions/deletions in the middle.

---

**Q6. What is ConcurrentHashMap and how is it thread-safe?**

> Java 8 ConcurrentHashMap uses **node-level locking** (synchronized on individual bucket's first node) instead of the table-level lock used by `Hashtable`. This allows concurrent reads and writes to different buckets. Read operations are **lock-free** (volatile reads). Write operations lock only the affected bucket.

---

**Q7. Explain the Java Memory Model (JMM) and the `volatile` keyword.**

> JMM defines how threads interact through memory. Without synchronization, threads cache values locally and may not see each other's updates (visibility problem). `volatile`:
> - Guarantees **visibility**: every write is immediately visible to all threads
> - Guarantees **ordering**: prevents instruction reordering around volatile reads/writes
> - Does NOT guarantee **atomicity** (use `AtomicInteger` for that)

---

**Q8. What is the difference between synchronized and ReentrantLock?**

| Feature | synchronized | ReentrantLock |
|---------|-------------|---------------|
| Unlock | Auto (block exit) | Must call `unlock()` in finally |
| Try lock | No | `tryLock(timeout)` — non-blocking |
| Fairness | No | `new ReentrantLock(true)` |
| Interruptible | No | `lockInterruptibly()` |
| Condition variables | `wait()/notify()` | Multiple `Condition` objects |

---

**Q9. What is a deadlock? How do you prevent it?**

> Deadlock occurs when thread A holds lock 1 and waits for lock 2, while thread B holds lock 2 and waits for lock 1.

> Prevention strategies:
> 1. **Lock ordering**: always acquire locks in the same global order
> 2. **Try-lock with timeout**: `lock.tryLock(timeout)` — back off if can't acquire
> 3. **Avoid nested locks**: minimize lock scope
> 4. **Database-level**: consistent ORDER BY on rows being locked

---

**Q10. Explain Java 8 Stream operations. Difference between intermediate and terminal?**

> **Intermediate operations** are lazy — they return a new Stream and are not evaluated until a terminal operation is called: `filter`, `map`, `flatMap`, `sorted`, `distinct`, `limit`, `peek`.

> **Terminal operations** trigger evaluation: `collect`, `forEach`, `count`, `findFirst`, `reduce`, `anyMatch`, `toList`.

```java
List<String> result = employees.stream()           // source
    .filter(e -> e.getSalary() > 50000)            // intermediate (lazy)
    .map(Employee::getName)                         // intermediate (lazy)
    .sorted()                                       // intermediate (lazy)
    .collect(Collectors.toList());                  // terminal (evaluates all above)
```

---

**Q11. What is the difference between `Comparable` and `Comparator`?**

> `Comparable` defines **natural ordering** — implemented by the class itself (`compareTo()`). `Comparator` defines **external ordering** — passed as a parameter or lambda. Use `Comparator` when the class doesn't implement `Comparable` or when you need multiple sort orders.

---

**Q12. What are the four types of references in Java?**

| Type | GC Behavior | Use Case |
|------|-------------|----------|
| Strong | Never collected while referenced | Default |
| Soft | Collected on memory pressure | Caches |
| Weak | Collected at next GC | WeakHashMap, listeners |
| Phantom | After finalization | Cleanup actions |

---

**Q13. Explain Java's Garbage Collection. What are the main algorithms?**

> Java GC reclaims memory occupied by unreachable objects. JVM generations:
> - **Young Generation** (Eden + Survivor spaces): most objects die here. Minor GC is fast.
> - **Old Generation**: long-lived objects. Major/Full GC is expensive.

> Algorithms:
> - **G1GC** (default Java 9+): divides heap into regions, concurrent marking, low pause
> - **ZGC** (Java 15+): sub-millisecond pauses, concurrent phases, scales to terabytes
> - **Serial/Parallel**: single/multi-threaded, stop-the-world — for simple/throughput workloads

---

**Q14. What is a `ThreadLocal` and when would you use it?**

> `ThreadLocal` provides thread-isolated storage — each thread has its own copy of the value. Use cases: per-request context (user ID, correlation ID, tenant), JDBC connections, `SimpleDateFormat` (which is not thread-safe).

> **Warning:** Must call `threadLocal.remove()` when done (especially in thread pools) to prevent memory leaks and data leaking between requests.

---

**Q15. What is the difference between `Runnable`, `Callable`, and `Future`?**

> `Runnable`: no return value, no checked exception. `Callable<V>`: returns a value, can throw checked exceptions. `Future<V>`: represents the result of an async computation — supports `get()` (blocking), `cancel()`, `isDone()`. `CompletableFuture` extends `Future` with composition: `thenApply`, `thenCompose`, `allOf`.

---

**Q16. What are functional interfaces? Name key ones from Java 8.**

> An interface with exactly one abstract method (can have default methods). Annotated with `@FunctionalInterface`.

| Interface | Signature | Example |
|-----------|-----------|---------|
| `Supplier<T>` | `() → T` | `() -> "hello"` |
| `Consumer<T>` | `T → void` | `System.out::println` |
| `Function<T,R>` | `T → R` | `String::length` |
| `Predicate<T>` | `T → boolean` | `s -> s.length() > 5` |
| `BiFunction<T,U,R>` | `(T,U) → R` | `(a,b) -> a + b` |
| `UnaryOperator<T>` | `T → T` | `String::toUpperCase` |

---

**Q17. What is Optional and why was it introduced?**

> `Optional<T>` is a container that may or may not hold a value — explicit alternative to `null`. Introduced to: reduce `NullPointerExceptions`, make "no value" explicit in API contracts. Correct usage: as method return type. Don't use as field type or method parameter.

```java
// Don't:  if (optional.get() != null) — get() throws NoSuchElementException!
// Do:     optional.ifPresent(v -> process(v))
// Do:     String result = optional.orElse("default")
// Do:     optional.orElseThrow(() -> new NotFoundException("x not found"))
```

---

**Q18. What is String interning? What is the String Pool?**

> String literals are stored in the **String Pool** (in Heap since Java 7). Same literal → same object reference. `new String("hello")` bypasses pool — creates new object. `intern()` adds a string to the pool and returns the pooled reference. `==` compares references; always use `.equals()` for String comparison.

---

**Q19. Explain the difference between `==` and `equals()`.**

> `==` compares **references** (memory addresses). `equals()` compares **content** (logical equality). For primitives, `==` is correct. For objects (String, Integer, etc.), always use `equals()`. `Integer` cache: `-128` to `127` are cached → `Integer.valueOf(127) == Integer.valueOf(127)` is true, `Integer.valueOf(128) == Integer.valueOf(128)` is false.

---

**Q20. What is `final`, `finally`, `finalize`?**

> - `final`: keyword — variable (constant), method (can't override), class (can't extend)
> - `finally`: block in try-catch that always executes (except `System.exit()`)
> - `finalize()`: deprecated method called by GC before object reclamation — don't use (unreliable, unpredictable timing)

---

**Q21. Explain covariance and contravariance in Java generics (`? extends T` vs `? super T`).**

> **`? extends T` (covariance)**: can read as T, cannot write. "Producer Extends" — use when you only read from the collection.

> **`? super T` (contravariance)**: can write T, cannot read (only Object). "Consumer Super" — use when you only write to the collection.

> **PECS**: Producer Extends, Consumer Super.

---

**Q22. What is type erasure in generics?**

> Java generics are compile-time only. At runtime, all generic type parameters are erased — `List<String>` becomes `List<Object>`. This means: no `instanceof List<String>`, no `new T[]`, generic type info not available at runtime (unless captured via reflection with `TypeToken`).

---

**Q23. What is the difference between checked and unchecked exceptions?**

> **Checked**: extends `Exception`. Must be declared in method signature (`throws`) or caught. Compile-time enforcement. Use for recoverable conditions (file not found, network error).

> **Unchecked**: extends `RuntimeException`. No compile-time requirement. Use for programming errors (null pointer, illegal argument, out of bounds).

> **Note**: `@Transactional` rolls back only for unchecked exceptions by default. Add `rollbackFor = Exception.class` to also rollback for checked exceptions.

---

**Q24. Explain `CompletableFuture`. Key methods?**

```java
// Non-blocking chaining
CompletableFuture.supplyAsync(() -> fetchUser(userId))          // run async
    .thenApply(user -> buildProfile(user))                       // transform result
    .thenCompose(profile -> fetchOrders(profile.getId()))        // async transform
    .thenAccept(orders -> sendEmail(orders))                     // terminal consume
    .exceptionally(ex -> { log.error(ex); return null; });       // error handling

// Combining
CompletableFuture.allOf(future1, future2).thenRun(() -> processAll());
CompletableFuture.anyOf(future1, future2).thenAccept(first -> processFirst(first));
```

---

**Q25. What are design patterns you use most in Spring applications?**

> **Singleton** (Spring beans), **Proxy** (AOP, `@Transactional`), **Template Method** (JdbcTemplate, RestTemplate), **Factory** (BeanFactory), **Strategy** (inject `Map<String, Strategy>` by bean name), **Observer** (ApplicationEvent), **Decorator** (filter chains), **Builder** (ResponseEntity.ok().body(...)).

---

# 2. Spring Boot & Spring Framework — Top 40 Questions

---

**Q26. What is IoC and Dependency Injection?**

> **IoC (Inversion of Control)**: the Spring container manages object creation and lifecycle — you describe dependencies, Spring provides them. **Dependency Injection**: the mechanism — Spring injects dependencies via constructor, setter, or field injection.

> Preferred: **Constructor injection** (immutable, testable, explicit dependencies). Avoid field injection in production code.

---

**Q27. What are the types of dependency injection?**

> 1. **Constructor injection** (recommended): `final` fields, immutable, fails fast on missing dependency
> 2. **Setter injection**: optional dependencies, can be changed later
> 3. **Field injection** (`@Autowired` on field): avoid in production — hard to test, hides dependencies

---

**Q28. Explain Spring Bean lifecycle.**

> 1. Container reads class/config
> 2. Instantiate bean (constructor)
> 3. Inject dependencies
> 4. Call `BeanNameAware.setBeanName()`
> 5. Call `BeanFactoryAware.setBeanFactory()`
> 6. Call `@PostConstruct` method
> 7. Call `InitializingBean.afterPropertiesSet()`
> 8. Call custom `initMethod`
> 9. Bean ready for use
> 10. On shutdown: `@PreDestroy` → `DisposableBean.destroy()` → custom `destroyMethod`

---

**Q29. What are Spring bean scopes?**

| Scope | Description |
|-------|-------------|
| `singleton` | One per ApplicationContext (default) |
| `prototype` | New instance per injection/request |
| `request` | One per HTTP request |
| `session` | One per HTTP session |
| `application` | One per ServletContext |
| `websocket` | One per WebSocket session |

> **Problem:** Injecting a `prototype` bean into a `singleton` bean — prototype is only created once. Fix: use `@Lookup` or `ObjectProvider<>`.

---

**Q30. How does Spring AOP work?**

> Spring AOP creates a **proxy** around the target bean. Two types:
> - **JDK Dynamic Proxy**: if bean implements an interface
> - **CGLIB Proxy**: if no interface (creates a subclass)
>
> **Limitation**: AOP doesn't work for self-invocation (calling a method on `this`) because `this` bypasses the proxy.

---

**Q31. What is `@Transactional` and how does it work internally?**

> `@Transactional` uses **AOP proxy**. When a method is called, the proxy:
> 1. Begins transaction (`connection.setAutoCommit(false)`)
> 2. Calls the actual method
> 3. Commits on success or rolls back on unchecked exception
>
> **Gotchas**: doesn't work with self-invocation; doesn't roll back for checked exceptions by default; `@Async` + `@Transactional` doesn't propagate transaction.

---

**Q32. What are the transaction propagation types?**

| Propagation | Behavior |
|-------------|----------|
| `REQUIRED` (default) | Join existing or create new |
| `REQUIRES_NEW` | Always create new, suspend existing |
| `NESTED` | Nested savepoint within existing |
| `SUPPORTS` | Join if exists, non-transactional if not |
| `NOT_SUPPORTED` | Suspend existing, run non-transactional |
| `MANDATORY` | Must have existing transaction, else exception |
| `NEVER` | Must not have transaction, else exception |

---

**Q33. Explain Spring Boot Auto-Configuration.**

> Spring Boot scans `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` and registers `@AutoConfiguration` classes. Each auto-config class uses `@ConditionalOnClass`, `@ConditionalOnMissingBean`, etc. to configure beans only when appropriate.

> Example: `DataSourceAutoConfiguration` creates a `DataSource` only if `javax.sql.DataSource` is on classpath AND no `DataSource` bean is already defined.

---

**Q34. What is Spring Actuator? Name key endpoints.**

> Actuator adds production-ready features to Spring Boot applications.

| Endpoint | Purpose |
|----------|---------|
| `/actuator/health` | Health status |
| `/actuator/metrics` | Micrometer metrics |
| `/actuator/prometheus` | Prometheus format metrics |
| `/actuator/env` | Configuration properties |
| `/actuator/loggers` | Change log levels at runtime |
| `/actuator/threaddump` | JVM thread dump |
| `/actuator/heapdump` | JVM heap dump |

---

**Q35. How does `@Cacheable` work in Spring?**

> Spring wraps the method in a proxy. On invocation: check cache for the key → if found (hit), return cached value without calling method. If miss: call method, store result in cache, return. `@CachePut` always calls the method AND updates cache. `@CacheEvict` removes from cache.

---

**Q36. How does Spring's exception handling work with `@RestControllerAdvice`?**

> `@RestControllerAdvice` is a global exception handler (combines `@ControllerAdvice` + `@ResponseBody`). Methods annotated with `@ExceptionHandler(SomeException.class)` are called when that exception propagates out of any controller. Spring searches for most-specific matching exception type.

---

**Q37. Explain `@Async`. How do you configure it?**

> Annotate method with `@Async`, enable with `@EnableAsync`. Spring wraps the method in an `AsyncTaskExecutor`. The method runs in a background thread and returns a `CompletableFuture` or `void`. Configure a custom `ThreadPoolTaskExecutor` bean for production (don't use default unbounded simple executor).

---

**Q38. What is the difference between `@Bean` and `@Component`?**

> `@Component` is for auto-detection (classpath scanning) — you own the class. `@Bean` is used in `@Configuration` classes to register third-party classes or when you need to configure the bean programmatically (e.g., configure a DataSource with custom properties from a library class you don't control).

---

**Q39. What is the difference between `@RestController` and `@Controller`?**

> `@RestController` = `@Controller` + `@ResponseBody`. With `@Controller`, you typically return view names. `@ResponseBody` (or `@RestController`) serializes the return value to JSON/XML and writes directly to the HTTP response body.

---

**Q40. How does Spring Security's filter chain work?**

> Spring Security adds a `DelegatingFilterProxy` to the servlet container, which delegates to `FilterChainProxy`. This holds a list of `SecurityFilterChain` objects. Each chain is matched to a request pattern and contains an ordered list of filters: authentication filters, session management, CSRF, authorization, etc.

---

**Q41. How do you implement custom validation?**

```java
// 1. Create annotation
@Constraint(validatedBy = PhoneValidator.class)
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface ValidPhone {
    String message() default "Invalid phone number";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

// 2. Implement validator
public class PhoneValidator implements ConstraintValidator<ValidPhone, String> {
    public boolean isValid(String value, ConstraintValidatorContext context) {
        return value != null && value.matches("^\\+?[1-9]\\d{1,14}$");
    }
}

// 3. Use
public class UserRequest {
    @ValidPhone
    private String phone;
}
```

---

# 3. Microservices & Distributed Systems — Top 30 Questions

---

**Q42. What is the difference between monolith and microservices?**

> Monolith: single deployable unit, shared database, simple transactions. Microservices: independently deployable services, database per service, network communication, eventual consistency. Choose microservices when teams are large, services need independent scaling, or have very different reliability needs.

---

**Q43. What is the Circuit Breaker pattern?**

> Monitors call failures. When failure rate exceeds threshold, "opens" the circuit — subsequent calls fail immediately (fast-fail) without hitting the failing service. After a wait period, enters "half-open" — sends test request. If successful, closes; if not, opens again. Prevents cascading failures.

---

**Q44. Explain the Saga pattern.**

> Manages distributed transactions using a sequence of local transactions with **compensating transactions** for rollback. Two styles:
> - **Choreography**: services react to events (event-driven, decoupled)
> - **Orchestration**: central coordinator directs steps (explicit flow, visible)

---

**Q45. What is the Outbox Pattern?**

> Solves dual-write problem (write to DB + publish to Kafka atomically). Solution: write event to an `outbox` table in the **same transaction** as the business operation. A separate poller or CDC tool (Debezium) reads the outbox table and publishes to Kafka. Ensures atomicity via DB transaction.

---

**Q46. What is CQRS?**

> Command Query Responsibility Segregation separates the model for writes (Commands → normalized DB, consistency) from reads (Queries → denormalized read model, performance). Enables independent scaling of read/write sides and optimized read models.

---

**Q47. How do you handle distributed transactions in microservices?**

> - **2PC (Two-Phase Commit)**: coordinator asks participants to prepare, then commit/rollback. Rarely used — blocking, coordinator SPOF, doesn't scale.
> - **Saga**: preferred. Local transactions + compensating transactions for rollback. Non-blocking, no SPOF, eventual consistency.

---

**Q48. What is service discovery? Client-side vs server-side?**

> **Service Discovery**: mechanism for services to find each other's addresses. **Client-side** (Eureka + Ribbon): client queries registry and load-balances. **Server-side** (AWS ALB, K8s Service): infrastructure routes transparently. Client-side: more flexible. Server-side: simpler client.

---

**Q49. What is the Bulkhead pattern?**

> Isolates resources (thread pools, connection pools) per downstream service. Prevents one slow service from consuming all resources and causing other services to fail. Like watertight compartments in a ship.

---

**Q50. Explain CAP theorem.**

> In a distributed system, you can only guarantee 2 of: Consistency (all nodes see same data), Availability (every request responds), Partition Tolerance (works despite network partition). Since partitions are inevitable, choose CP (consistency sacrificed for availability) or AP (availability over consistency).

---

**Q51. What is eventual consistency?**

> In distributed systems, after a write, different nodes may temporarily show different values. Eventually, all nodes will converge to the same value — but there's a window of inconsistency. Acceptable for product catalogs, social feeds. Not acceptable for bank balances, inventory.

---

# 4. Apache Kafka — Top 25 Questions

---

**Q52. What is Kafka and how does it differ from RabbitMQ?**

> Kafka is a **distributed commit log** optimized for high-throughput, durable event streaming. Messages retained after consumption (configurable). Multiple consumer groups read independently. Supports replay.

> RabbitMQ is a **message broker** — messages deleted after consumption, push-based, simpler routing (exchanges/bindings), lower throughput.

---

**Q53. What is a partition and why is it important?**

> Partition is the basic unit of parallelism in Kafka. A topic is split into partitions — each is an ordered log. Consumers in a group each get a subset of partitions → parallel processing. Ordering guaranteed within a partition, not across partitions.

---

**Q54. How does Kafka achieve high throughput?**

> 1. **Sequential disk I/O**: append-only log, leverages OS page cache
> 2. **Batching**: producers batch messages; brokers write in bulk
> 3. **Zero-copy**: `sendfile()` syscall sends data from disk to network socket without copying to user space
> 4. **Compression**: batch-level compression (snappy, lz4, gzip)
> 5. **Pull-based consumers**: consumers control their rate

---

**Q55. Explain the three delivery guarantees.**

> **At-most-once**: message may be lost, never duplicated. `acks=0`, auto-commit. **At-least-once**: no loss, may be duplicated. `acks=all`, manual commit + consumer idempotency. **Exactly-once**: no loss, no duplicates. Idempotent producer + transactional API + `read_committed` isolation.

---

**Q56. What is consumer group rebalancing? When does it happen?**

> Rebalancing reassigns partition ownership among consumers in a group when: consumer joins, consumer leaves/crashes, partitions added. During classic rebalancing, all consumers stop processing (stop-the-world). **Cooperative Sticky Assignor** minimizes disruption by reassigning only necessary partitions incrementally.

---

**Q57. How do you handle a poison pill message?**

> Configure `DeadLetterPublishingRecoverer` with `DefaultErrorHandler`. Retry N times with backoff. After exhausting retries, publish to `{topic}.DLT`. Consumer on DLT topic investigates, fixes, and replays or discards. Mark `DeserializationException` as non-retryable.

---

**Q58. How does the idempotent producer work?**

> Enable with `enable.idempotence=true`. Broker assigns a **Producer ID (PID)** and tracks a **sequence number per partition**. If broker receives a duplicate message (same PID + sequence), it deduplicates silently. Prevents duplicates from producer retries.

---

**Q59. What is log compaction?**

> Instead of time-based retention, Kafka can retain only the **latest value per key** for a topic (`cleanup.policy=compact`). Useful for event-sourced state — consumers can rebuild current state from the compacted log. Deleted keys get a **tombstone** record (null value).

---

**Q60. How would you tune Kafka consumer throughput?**

> 1. Increase `concurrency` (= more consumer threads, up to partition count)
> 2. Use batch listener with `max.poll.records` tuning
> 3. Increase `fetch.min.bytes` and `fetch.max.wait.ms` for larger fetches
> 4. Process async within the listener (with careful ack management)
> 5. Optimize processing logic (batch DB inserts, async I/O)

---

# 5. Databases & SQL — Top 25 Questions

---

**Q61. What is an index? How does a B-Tree index work?**

> Index is a data structure that speeds up lookups. B-Tree (Balanced Tree) index: tree with sorted keys, O(log n) lookup. Leaf nodes contain key + row pointer (heap tuple). Index scan: traverse tree to find key, follow pointer to table. Useful for equality, range, prefix queries.

---

**Q62. What is the N+1 query problem?**

> Fetching N parent entities results in N additional queries to fetch children. Example: fetch 100 orders → 100 separate queries for each order's items. Fix: `JOIN FETCH` or `@EntityGraph` in JPA, `IN` clause, or batch fetching.

---

**Q63. What is a database transaction? What are ACID properties?**

> **Atomicity**: all-or-nothing. **Consistency**: database remains in valid state. **Isolation**: concurrent transactions appear sequential. **Durability**: committed data survives crashes (WAL). Implemented via: WAL (durability), locks/MVCC (isolation), rollback logs (atomicity).

---

**Q64. Explain database isolation levels.**

| Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|-------|-----------|--------------------|-----------------------------|
| READ UNCOMMITTED | ✓ possible | ✓ possible | ✓ possible |
| READ COMMITTED | ✗ prevented | ✓ possible | ✓ possible |
| REPEATABLE READ | ✗ | ✗ prevented | ✓ possible |
| SERIALIZABLE | ✗ | ✗ | ✗ prevented |

> PostgreSQL default: READ COMMITTED. MySQL InnoDB default: REPEATABLE READ (uses MVCC).

---

**Q65. What is MVCC?**

> Multi-Version Concurrency Control: readers don't block writers, writers don't block readers. Each row has hidden columns (`xmin`, `xmax`) storing the transaction ID that created/deleted it. Each transaction sees a snapshot of the DB from its start time. No read locks needed.

---

**Q66. Explain optimistic vs pessimistic locking.**

> **Pessimistic** (`SELECT FOR UPDATE`): acquires lock immediately, prevents concurrent modification. Suitable when conflict probability is high. **Optimistic** (`@Version`): no lock during read; on update, verify version matches (if not, throw `OptimisticLockException`). Suitable when conflict probability is low. Better concurrency but requires retry on conflict.

---

**Q67. What is database partitioning vs sharding?**

> **Partitioning**: split a large table into smaller tables within the **same database instance** (Range, List, Hash). Managed by the DB. **Sharding**: split data across **multiple database servers**. Managed by application or middleware. Sharding enables horizontal scaling beyond single-server limits.

---

**Q68. What is the difference between DELETE, TRUNCATE, and DROP?**

| | DELETE | TRUNCATE | DROP |
|--|--------|----------|------|
| Removes | Selected rows | All rows | Entire table |
| WHERE clause | Yes | No | No |
| Rollback | Yes (transactional) | Depends (Postgres: yes; MySQL: no) | No |
| Triggers fired | Yes | No | No |
| Speed | Slow (row-by-row) | Fast (deallocate pages) | Instant |

---

**Q69. How do you identify and fix a slow query?**

> 1. Enable slow query log (`log_min_duration_statement = 1000` in PostgreSQL)
> 2. Find offender in `pg_stat_statements` (order by `total_exec_time`)
> 3. Run `EXPLAIN (ANALYZE, BUFFERS)` on the query
> 4. Look for: Sequential Scan on large table, Nested Loop with huge rows, high buffer reads
> 5. Add missing index, rewrite query, update statistics (`ANALYZE`)
> 6. Verify improvement: re-run EXPLAIN

---

**Q70. What is connection pooling? Why is it needed?**

> Creating a DB connection is expensive (TLS handshake, authentication, ~50ms). Connection pool maintains pre-created connections and reuses them. HikariCP is the fastest Java connection pool. Key settings: `maximumPoolSize` (rule: `cores × 2 + drives` or `db_max_connections / num_pods`).

---

# 6. Redis — Top 20 Questions

---

**Q71. Why is Redis faster than a relational database?**

> 1. **In-memory**: data in RAM, no disk I/O for reads
> 2. **Simple data structures**: no query parsing, no joins
> 3. **Single-threaded command processing**: no lock contention (I/O multiplexing for connections)
> 4. **Non-blocking I/O**: handles thousands of concurrent connections efficiently

---

**Q72. How does Redis handle persistence?**

> **RDB (Redis Database Snapshot)**: periodic point-in-time snapshots to disk. Fast restore, but data loss between snapshots. **AOF (Append Only File)**: log every write command. Up to date, but larger file, slower restart. **RDB+AOF**: both (recommended for production — AOF for durability, RDB for fast restart).

---

**Q73. What is the difference between Redis Cluster and Sentinel?**

> **Sentinel**: HA for a single master. Monitors master, auto-failover to replica. No horizontal scaling (one master handles all writes). **Cluster**: shards data across multiple masters (16384 hash slots). Horizontal scaling for write throughput. More complex, limited multi-key operations.

---

**Q74. How would you implement a distributed lock in Redis?**

> `SET key value NX EX 30` — atomic: set if not exists, expire in 30 seconds. Value = unique ID (UUID). Release using Lua script to atomically check value before deleting (prevent releasing another thread's lock). Use Redisson in production for auto-renewal and fair locking.

---

**Q75. What is Cache Stampede and how do you prevent it?**

> When a cached item expires, many concurrent requests miss the cache and all hit the database simultaneously. Prevention: **mutex** (only one goroutine refreshes, others wait), **probabilistic early expiration** (proactively refresh before TTL expires), **background refresh**.

---

# 7. System Design — Top 15 Questions

---

**Q76. Design a URL shortener that handles 1 billion redirects per day.**

> **Scale**: 11K read QPS, 1.2K write QPS. **Short code**: Base62 encoding of auto-increment ID (no collisions). **DB**: PostgreSQL with index on `short_code`. **Cache**: Redis for top 20% URLs (80% traffic cache hit). **Scaling**: Multiple read replicas, cache layer absorbs most traffic. See Section 11 for full design.

---

**Q77. How would you design a notification service for 100 million users?**

> Event-driven architecture: upstream services publish events to Kafka. Notification service consumes events, fan-outs to individual user notifications (batch to Kafka). Channel workers (email, SMS, push) consume and send. Deduplication via Redis SETNX with event ID. Rate limiting per user. DLT for failed sends.

---

**Q78. How would you handle API rate limiting at scale?**

> Redis sliding window with Sorted Sets. Each key = `rate_limit:{userId}`. Lua script: remove expired entries, count remaining, increment if under limit. Fail-open (allow if Redis unavailable). Return `429 Too Many Requests` with `Retry-After` header. See Section 10.

---

**Q79. How do you scale a database that's becoming a bottleneck?**

> 1. **Read replicas**: route read-only queries to replicas
> 2. **Caching**: Redis for frequently-read, rarely-changed data
> 3. **Connection pooling**: ensure efficient connection reuse
> 4. **Query optimization**: indexes, EXPLAIN ANALYZE
> 5. **Vertical scaling**: bigger machine
> 6. **Partitioning**: split large tables within same DB
> 7. **Sharding**: split data across multiple DB servers

---

# 8. Behavioral & Situational — Top 15 Questions

---

**Q80. Tell me about a production incident you handled.**

> **Structure (STAR):**
> - **Situation**: what was the system and what broke?
> - **Task**: what was your role?
> - **Action**: what steps did you take?
> - **Result**: what was the outcome? What did you learn?
>
> **Sample**: Payment service had 5% error rate. Checked dashboards, found DB CPU at 98%. Used `pg_stat_statements` to identify a missing index on `payments.user_id`. Added index concurrently (no downtime). Error rate dropped to 0.1% in 2 minutes.

---

**Q81. How do you ensure code quality in a team setting?**

> Code reviews (PR checklist), automated tests (JUnit, Testcontainers for integration), static analysis (SonarQube), linting (Checkstyle), coverage gates in CI/CD, pair programming for complex features, architecture decision records (ADRs) for major decisions.

---

**Q82. How do you approach debugging a performance issue?**

> 1. Define the baseline and measure (what's the expected vs actual latency/throughput?)
> 2. Profile (JProfiler, Async Profiler) — find the hotspot
> 3. Check externals first (DB slow queries, cache hit rate, network latency)
> 4. Fix the bottleneck, measure again
> 5. Document root cause and prevention

---

**Q83. How do you handle technical debt?**

> Track in a dedicated backlog, not lost in comments. Categorize by risk and effort. Negotiate time in sprints (20% rule). Refactor incrementally alongside feature work. Use strangler fig pattern for large rewrites.

---

**Q84. Describe your experience with microservices. What challenges have you faced?**

> Focus on: distributed transactions (chose Saga over 2PC), service discovery (Eureka, K8s DNS), debugging across services (distributed tracing with correlation IDs), eventual consistency (idempotent consumers), infrastructure complexity (Kubernetes deployments). Show that you understand the real operational costs, not just the theoretical benefits.

---

## Quick Reference — Common Mistakes to Avoid in Interviews

```
Core Java Mistakes:
  ✗ Saying == can compare String content
  ✗ Confusing HashMap thread-safety (it's not thread-safe)
  ✗ Saying GC runs on demand (it's JVM controlled)
  ✗ Forgetting volatile doesn't guarantee atomicity

Spring Mistakes:
  ✗ Saying @Transactional works on private methods
  ✗ Forgetting self-invocation bypasses AOP proxy
  ✗ Not knowing that checked exceptions don't auto-rollback
  ✗ Saying @Autowired is the only way to inject (constructor is preferred)

Microservices Mistakes:
  ✗ Proposing 2PC for distributed transactions (use Saga instead)
  ✗ Not mentioning idempotency when discussing Saga/events
  ✗ Saying microservices always better than monolith
  ✗ Forgetting that DB per service prevents direct JOINs

Kafka Mistakes:
  ✗ Saying Kafka guarantees ordering globally (only within partition)
  ✗ Not mentioning idempotency for at-least-once consumers
  ✗ Forgetting that more consumers than partitions = idle consumers

SQL Mistakes:
  ✗ Not knowing EXPLAIN output
  ✗ Confusing index not used when function on column
  ✗ Forgetting that OFFSET pagination becomes slow on large offsets
  ✗ Not mentioning MVCC when explaining isolation levels
```
