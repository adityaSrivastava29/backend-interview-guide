---
layout: default
title: Redis
nav_order: 11
---

# Section 10: Redis — Caching, Data Structures & Distributed Patterns

> Data structures, caching strategies, distributed locks, and rate limiting

---

## Table of Contents

1. [Redis Data Structures](#1-redis-data-structures)
2. [Spring Boot Redis Integration](#2-spring-boot-redis-integration)
3. [Caching Strategies](#3-caching-strategies)
4. [Eviction Policies](#4-eviction-policies)
5. [Distributed Locking](#5-distributed-locking)
6. [Rate Limiting](#6-rate-limiting)
7. [Production Problems & Solutions](#7-production-problems--solutions)
8. [Redis Deployment Topologies](#8-redis-deployment-topologies)
9. [Interview Questions](#9-interview-questions)

---

# 1. Redis Data Structures

## String

Most versatile. Store JSON, numbers, binary data.

```redis
SET user:1001 "{\"name\":\"Alice\",\"email\":\"alice@example.com\"}"
GET user:1001

# Atomic increment (counters)
INCR page:views:home          # atomic: safe for concurrent updates
INCRBY page:views:home 5

# Set with TTL
SET session:abc123 "{...}" EX 3600  # expires in 3600 seconds
SET session:abc123 "{...}" PX 5000  # expires in 5000 milliseconds

# SETNX: Set if Not eXists (foundation of distributed lock)
SET lock:order:1001 "process-1" NX EX 30  # NX=only set if not exists
```

## List

Ordered, allows duplicates. Push/pop from either end.

```redis
LPUSH queue:emails "email1" "email2"  # push to left
RPUSH queue:emails "email3"           # push to right
RPOP queue:emails                     # pop from right (FIFO queue)
LPOP queue:emails                     # pop from left (Stack)
LRANGE queue:emails 0 -1              # get all elements
LLEN queue:emails                     # length

# Blocking pop — wait up to 30s for an element (worker queue pattern)
BRPOP queue:emails 30
```

## Set

Unordered, unique elements. Fast membership tests.

```redis
SADD user:1001:interests "java" "spring" "kafka"
SISMEMBER user:1001:interests "java"   # 1 (true)
SISMEMBER user:1001:interests "php"    # 0 (false)
SMEMBERS user:1001:interests           # get all
SCARD user:1001:interests              # count

# Set operations (useful for social features)
SUNION user:1001:interests user:1002:interests   # union
SINTER user:1001:interests user:1002:interests   # intersection (common interests)
SDIFF user:1001:interests user:1002:interests    # difference
```

## Sorted Set (ZSet)

Each element has a **score** (float). Elements ordered by score. Best for leaderboards, ranges.

```redis
ZADD leaderboard 1500 "alice"
ZADD leaderboard 2300 "bob"
ZADD leaderboard 1800 "charlie"

ZRANK leaderboard "alice"         # 0 (rank by score ascending)
ZREVRANK leaderboard "bob"        # 0 (rank by score descending → #1)
ZSCORE leaderboard "bob"          # 2300.0
ZRANGE leaderboard 0 2 WITHSCORES  # bottom 3
ZREVRANGE leaderboard 0 2 WITHSCORES # top 3

# Range by score (e.g., users with 1000-2000 points)
ZRANGEBYSCORE leaderboard 1000 2000

# Use for sliding window rate limiting!
```

## Hash

Map of field-value pairs. Like a row in a database.

```redis
HSET product:5001 name "MacBook Pro" price 1999.99 stock 50 category "laptop"
HGET product:5001 price          # "1999.99"
HMGET product:5001 name price    # ["MacBook Pro", "1999.99"]
HGETALL product:5001             # all fields
HINCRBY product:5001 stock -1   # atomic stock decrement

# Better than storing as JSON string when you need partial updates
# JSON String: GET → deserialize → update field → serialize → SET (entire object)
# Hash: HINCRBY product:5001 stock -1 (atomic, no read-modify-write)
```

## Stream

Append-only log. Similar to Kafka topic but within Redis.

```redis
XADD orders * orderId 1001 amount 99.99 userId 501
XADD orders * orderId 1002 amount 149.99 userId 502

XREAD COUNT 10 STREAMS orders 0         # read from beginning
XREAD BLOCK 1000 STREAMS orders $       # wait for new entries

# Consumer groups (like Kafka consumer groups)
XGROUP CREATE orders inventory-service $ MKSTREAM
XREADGROUP GROUP inventory-service consumer-1 COUNT 10 STREAMS orders >
XACK orders inventory-service <message-id>   # acknowledge processing
```

---

# 2. Spring Boot Redis Integration

## Setup

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

```yaml
spring:
  data:
    redis:
      host: redis
      port: 6379
      password: ${REDIS_PASSWORD}
      lettuce:
        pool:
          max-active: 20    # max connections in pool
          max-idle: 10
          min-idle: 5
          max-wait: 1000ms
        # Cluster mode
        # cluster:
        #   nodes: redis1:6379,redis2:6379,redis3:6379
```

## RedisTemplate

```java
@Configuration
public class RedisConfig {
    
    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setHashValueSerializer(new GenericJackson2JsonRedisSerializer());
        return template;
    }
}

@Service
public class ProductCacheService {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    private static final String PRODUCT_KEY_PREFIX = "product:";
    private static final Duration CACHE_TTL = Duration.ofHours(1);
    
    public void cacheProduct(Product product) {
        String key = PRODUCT_KEY_PREFIX + product.getId();
        redisTemplate.opsForValue().set(key, product, CACHE_TTL);
    }
    
    public Optional<Product> getCachedProduct(Long productId) {
        String key = PRODUCT_KEY_PREFIX + productId;
        Product product = (Product) redisTemplate.opsForValue().get(key);
        return Optional.ofNullable(product);
    }
    
    public void evictProduct(Long productId) {
        redisTemplate.delete(PRODUCT_KEY_PREFIX + productId);
    }
    
    // Hash operations
    public void updateProductStock(Long productId, int quantity) {
        String key = PRODUCT_KEY_PREFIX + productId;
        redisTemplate.opsForHash().increment(key, "stock", quantity);
    }
    
    // Sorted set (leaderboard)
    public void updateScore(String userId, double score) {
        redisTemplate.opsForZSet().add("leaderboard", userId, score);
    }
    
    public Set<ZSetOperations.TypedTuple<Object>> getTopPlayers(int count) {
        return redisTemplate.opsForZSet()
            .reverseRangeWithScores("leaderboard", 0, count - 1);
    }
}
```

## Spring Cache Annotations (Cache Abstraction)

```java
@Configuration
@EnableCaching
public class CacheConfig {
    
    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
        // Default TTL configuration
        RedisCacheConfiguration defaultConfig = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .disableCachingNullValues()
            .serializeKeysWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new GenericJackson2JsonRedisSerializer()));
        
        // Per-cache TTL overrides
        Map<String, RedisCacheConfiguration> cacheConfigs = Map.of(
            "products", defaultConfig.entryTtl(Duration.ofHours(1)),
            "sessions", defaultConfig.entryTtl(Duration.ofMinutes(30)),
            "rates", defaultConfig.entryTtl(Duration.ofSeconds(60))
        );
        
        return RedisCacheManager.builder(factory)
            .cacheDefaults(defaultConfig)
            .withInitialCacheConfigurations(cacheConfigs)
            .build();
    }
}

@Service
public class ProductService {
    
    @Cacheable(value = "products", key = "#productId",
               condition = "#productId > 0",
               unless = "#result == null")
    public Product findById(Long productId) {
        // Only called if not in cache
        return productRepository.findById(productId).orElseThrow();
    }
    
    @CachePut(value = "products", key = "#product.id")
    public Product update(Product product) {
        // Always executes AND updates cache
        return productRepository.save(product);
    }
    
    @CacheEvict(value = "products", key = "#productId")
    public void delete(Long productId) {
        // Removes from cache after deleting
        productRepository.deleteById(productId);
    }
    
    @CacheEvict(value = "products", allEntries = true)
    public void clearAllProducts() {
        // Clears entire "products" cache (use sparingly!)
    }
}
```

---

# 3. Caching Strategies

## Cache-Aside (Lazy Loading) — Most Common

```
Application reads:
1. Check cache → cache miss?
2. Query database
3. Populate cache
4. Return data

Cache is populated on demand. 
Cold start: first request hits DB (cache miss).
```

```java
public Product getProduct(Long id) {
    // Cache-aside pattern
    return cache.get("product:" + id, Product.class)
        .orElseGet(() -> {
            Product product = productRepository.findById(id).orElseThrow();
            cache.set("product:" + id, product, Duration.ofHours(1));
            return product;
        });
}
```

## Write-Through

```
Application writes:
1. Write to cache
2. Write to database (synchronously)

Cache always consistent with DB.
Write penalty: every write hits both cache and DB.
Use when: reads are much more frequent than writes, data must be fresh.
```

## Write-Behind (Write-Back)

```
Application writes:
1. Write to cache (fast, async ACK)
2. Background process writes to database (async, batched)

Better write throughput. Risk: data loss if cache fails before DB write.
Use when: extremely high write rate, DB is the bottleneck.
```

## Read-Through

```
Cache manages DB reads:
1. App asks cache → cache misses?
2. Cache queries DB directly
3. Cache stores result
4. Returns to app

App doesn't know about DB.
Requires cache that supports read-through (e.g., Hazelcast).
```

---

# 4. Eviction Policies

When Redis reaches `maxmemory`, it must evict keys. Configure via `maxmemory-policy`.

| Policy | What Gets Evicted | Use When |
|--------|-------------------|----------|
| `noeviction` | Error returned (no eviction) | Critical data, no eviction OK |
| `allkeys-lru` | Least Recently Used from all keys | General-purpose cache |
| `volatile-lru` | LRU from keys with TTL set | Mix of cache + persistent data |
| `allkeys-lfu` | Least Frequently Used from all keys | Skewed access patterns |
| `volatile-lfu` | LFU from keys with TTL | Skewed access + persistent data |
| `allkeys-random` | Random from all keys | Uniform access distribution |
| `volatile-random` | Random from keys with TTL | |
| `volatile-ttl` | Keys with shortest TTL remaining | |

**Recommended for most caches:** `allkeys-lru`

```yaml
spring:
  data:
    redis:
      host: redis
      
# redis.conf
maxmemory 2gb
maxmemory-policy allkeys-lru
```

---

# 5. Distributed Locking

## Why Needed

```
Without distributed lock:
  Thread 1 (Pod A): Check stock → 1 unit → Reserve → Sell
  Thread 2 (Pod B): Check stock → 1 unit → Reserve → OVERSELL!

With distributed lock:
  Thread 1 (Pod A): ACQUIRE LOCK → Check stock → Reserve → RELEASE LOCK
  Thread 2 (Pod B): ACQUIRE LOCK → [waits] → Gets lock → Check stock → 0 units → Return error
```

## SET NX (Simple Lock)

```java
@Service
public class SimpleRedisLock {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    private static final Duration LOCK_TIMEOUT = Duration.ofSeconds(30);
    
    public boolean tryLock(String lockKey, String lockValue) {
        // SET key value NX EX 30  (atomic: set only if not exists, expire in 30s)
        Boolean acquired = redisTemplate.opsForValue()
            .setIfAbsent(lockKey, lockValue, LOCK_TIMEOUT);
        return Boolean.TRUE.equals(acquired);
    }
    
    public void unlock(String lockKey, String lockValue) {
        // MUST check value before deleting — only owner can release!
        // Use Lua script for atomicity
        String luaScript = """
            if redis.call('get', KEYS[1]) == ARGV[1] then
                return redis.call('del', KEYS[1])
            else
                return 0
            end
            """;
        redisTemplate.execute(
            new DefaultRedisScript<>(luaScript, Long.class),
            List.of(lockKey),
            lockValue
        );
    }
    
    public <T> T withLock(String lockKey, Duration timeout, Supplier<T> operation) {
        String lockValue = UUID.randomUUID().toString();
        
        if (!tryLock(lockKey, lockValue)) {
            throw new LockAcquisitionException("Could not acquire lock: " + lockKey);
        }
        
        try {
            return operation.get();
        } finally {
            unlock(lockKey, lockValue);
        }
    }
}

// Usage
public OrderResult placeOrder(String productId, int quantity) {
    return redisLock.withLock("lock:product:" + productId, Duration.ofSeconds(10), () -> {
        // Critical section: check stock and reserve atomically
        int currentStock = inventoryService.getStock(productId);
        if (currentStock < quantity) {
            throw new InsufficientStockException();
        }
        inventoryService.reduceStock(productId, quantity);
        return orderService.createOrder(productId, quantity);
    });
}
```

## Redisson (Production Distributed Lock)

```xml
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson-spring-boot-starter</artifactId>
    <version>3.23.4</version>
</dependency>
```

```java
@Service
public class RedissonLockService {
    
    @Autowired
    private RedissonClient redissonClient;
    
    public void processCriticalSection(String resourceId) throws InterruptedException {
        RLock lock = redissonClient.getLock("lock:" + resourceId);
        
        // Try lock with timeout: wait 5s, auto-release after 30s
        boolean acquired = lock.tryLock(5, 30, TimeUnit.SECONDS);
        
        if (!acquired) {
            throw new LockAcquisitionException("Resource locked: " + resourceId);
        }
        
        try {
            // Critical section
            processResource(resourceId);
        } finally {
            lock.unlock();
        }
    }
    
    // Fair lock: respects order of acquisition requests
    public void fairLockExample(String resourceId) {
        RLock fairLock = redissonClient.getFairLock("fairlock:" + resourceId);
        // ...
    }
    
    // Read-Write lock: multiple readers, one writer
    public void readWriteLockExample(String resourceId) {
        RReadWriteLock rwLock = redissonClient.getReadWriteLock("rwlock:" + resourceId);
        
        // Multiple readers can hold simultaneously
        rwLock.readLock().lock();
        try { readResource(resourceId); }
        finally { rwLock.readLock().unlock(); }
        
        // Only one writer, blocks readers
        rwLock.writeLock().lock();
        try { updateResource(resourceId); }
        finally { rwLock.writeLock().unlock(); }
    }
}
```

---

# 6. Rate Limiting

## Token Bucket with Redis

```java
@Service
public class RateLimiter {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    // Sliding window rate limiter using sorted set
    // Key: rate_limit:{userId}  Value: sorted set of timestamps
    public boolean isAllowed(String userId, int maxRequests, Duration window) {
        String key = "rate_limit:" + userId;
        long now = System.currentTimeMillis();
        long windowStart = now - window.toMillis();
        
        String luaScript = """
            local key = KEYS[1]
            local now = tonumber(ARGV[1])
            local window_start = tonumber(ARGV[2])
            local max_requests = tonumber(ARGV[3])
            local ttl = tonumber(ARGV[4])
            
            -- Remove expired entries (older than window)
            redis.call('zremrangebyscore', key, '-inf', window_start)
            
            -- Count current requests in window
            local current = redis.call('zcard', key)
            
            if current < max_requests then
                -- Add current request with timestamp as score
                redis.call('zadd', key, now, now .. '-' .. math.random())
                redis.call('expire', key, ttl)
                return 1  -- allowed
            else
                return 0  -- denied
            end
            """;
        
        Long result = redisTemplate.execute(
            new DefaultRedisScript<>(luaScript, Long.class),
            List.of(key),
            String.valueOf(now),
            String.valueOf(windowStart),
            String.valueOf(maxRequests),
            String.valueOf(window.getSeconds() + 1)
        );
        
        return Long.valueOf(1).equals(result);
    }
}

// Spring API rate limiting
@Component
public class RateLimitInterceptor implements HandlerInterceptor {
    
    @Autowired
    private RateLimiter rateLimiter;
    
    @Override
    public boolean preHandle(HttpServletRequest request,
                              HttpServletResponse response,
                              Object handler) {
        
        String userId = (String) request.getAttribute("userId");
        
        if (!rateLimiter.isAllowed(userId, 100, Duration.ofMinutes(1))) {
            response.setStatus(429); // Too Many Requests
            response.setHeader("X-RateLimit-Limit", "100");
            response.setHeader("Retry-After", "60");
            return false;
        }
        return true;
    }
}
```

---

# 7. Production Problems & Solutions

## Problem 1: Cache Stampede (Thundering Herd)

**Scenario:** Popular product's cache entry expires at 3am. Thousands of simultaneous requests all hit DB at once.

```
T=0: 10,000 users request /products/1001
Cache key "product:1001" expired 1ms ago
→ All 10,000 requests simultaneously query the database
→ Database overwhelmed, query time goes from 10ms to 5 seconds
→ Your monitoring alert fires
```

**Solution 1: Probabilistic Early Expiration**
```java
// Don't wait for TTL to expire — proactively refresh before expiry
public Product getProduct(Long id) {
    ProductWithMeta cached = cache.get("product:" + id);
    
    if (cached != null) {
        long remainingTtl = cached.getRemainingTtlMs();
        long totalTtl = 3600_000L; // 1 hour in ms
        
        // 10% chance of proactive refresh when 10% TTL remains
        double probability = Math.exp(-0.01 * remainingTtl / totalTtl * Math.log(10));
        if (remainingTtl > 60_000 || Math.random() > probability) {
            return cached.getProduct();
        }
        // Fall through to refresh
    }
    
    // Refresh cache
    Product fresh = productRepository.findById(id).orElseThrow();
    cache.set("product:" + id, fresh, Duration.ofHours(1));
    return fresh;
}
```

**Solution 2: Mutex (Distributed Lock on Cache Miss)**
```java
public Product getProduct(Long id) {
    Product cached = cache.get("product:" + id);
    if (cached != null) return cached;
    
    String lockKey = "cache_lock:product:" + id;
    
    // Only ONE thread refreshes the cache, others wait
    if (redisLock.tryLock(lockKey, UUID.randomUUID().toString())) {
        try {
            // Double-check after acquiring lock (another thread might have populated)
            cached = cache.get("product:" + id);
            if (cached != null) return cached;
            
            Product fresh = productRepository.findById(id).orElseThrow();
            cache.set("product:" + id, fresh, Duration.ofHours(1));
            return fresh;
        } finally {
            redisLock.unlock(lockKey);
        }
    } else {
        // Another thread is refreshing — wait and return stale or empty
        Thread.sleep(100);
        return cache.getOrDefault("product:" + id, getDefaultProduct(id));
    }
}
```

## Problem 2: Cache Penetration

**Scenario:** Attacker sends requests for IDs that don't exist (e.g., /products/-1, /products/99999999). Each request misses cache, hits DB, returns null. DB overwhelmed.

**Solution 1: Cache Null Values**
```java
public Product getProduct(Long id) {
    String cacheKey = "product:" + id;
    
    Object cached = cache.get(cacheKey);
    if (cached != null) {
        return cached == NULL_SENTINEL ? null : (Product) cached;
    }
    
    Product product = productRepository.findById(id).orElse(null);
    
    if (product == null) {
        // Cache "not found" for 5 minutes (short TTL)
        cache.set(cacheKey, NULL_SENTINEL, Duration.ofMinutes(5));
        return null;
    }
    
    cache.set(cacheKey, product, Duration.ofHours(1));
    return product;
}
```

**Solution 2: Bloom Filter**
```java
@Service
public class ProductCacheService {
    
    // Bloom filter: fast probabilistic data structure
    // "Is product ID 12345 in our database?" → NO (definite) or MAYBE (check DB)
    private final RBloomFilter<Long> productExistenceFilter;
    
    public ProductCacheService(RedissonClient redisson) {
        this.productExistenceFilter = redisson.getBloomFilter("product-existence");
        productExistenceFilter.tryInit(10_000_000L, 0.03); // 10M items, 3% false positive
    }
    
    // Call on startup / when new products added
    public void addProductToBloomFilter(Long productId) {
        productExistenceFilter.add(productId);
    }
    
    public Product getProduct(Long id) {
        // Bloom filter: definite NO → skip DB entirely
        if (!productExistenceFilter.contains(id)) {
            return null; // definitely doesn't exist
        }
        // MAYBE → check cache then DB
        return cache.get("product:" + id, () -> productRepository.findById(id).orElse(null));
    }
}
```

## Problem 3: Cache Avalanche

**Scenario:** All cache keys set with same TTL expire simultaneously. Massive DB load spike.

**Solution: TTL Jitter**
```java
// Bad: all products expire at the same time
cache.set("product:" + id, product, Duration.ofHours(1));

// Good: add random jitter (±20% of TTL)
Duration baseTtl = Duration.ofHours(1);
long jitterMs = (long) (baseTtl.toMillis() * 0.2 * (Math.random() * 2 - 1));
Duration ttl = baseTtl.plusMillis(jitterMs);  // 48–72 minutes

cache.set("product:" + id, product, ttl);
```

---

# 8. Redis Deployment Topologies

## Standalone
```
Single Redis instance. No HA. For development only.
```

## Redis Sentinel (High Availability)

```
┌─────────────┐  ┌─────────────────┐  ┌─────────────┐
│ Sentinel 1  │  │   Sentinel 2    │  │ Sentinel 3  │
└──────┬──────┘  └────────┬────────┘  └──────┬──────┘
       │                  │                  │
       └──────────────────┼──────────────────┘
                          │ Monitor + Failover
              ┌───────────┴───────────────┐
              │                           │
    ┌─────────┴──────┐         ┌──────────┴─────┐
    │  Master Redis  │─────→──→│  Replica Redis  │
    │ (read + write) │ replicate│  (read only)   │
    └────────────────┘         └────────────────┘
    
If master dies → Sentinels vote → promote replica to master
```

```yaml
spring:
  data:
    redis:
      sentinel:
        master: mymaster
        nodes: sentinel1:26379,sentinel2:26379,sentinel3:26379
```

## Redis Cluster (Horizontal Scaling)

```
16384 hash slots distributed across N master nodes.
Each master has 1+ replicas.
Data sharded automatically by key hash.

Node 1 (Master): slots 0 - 5460       + Replica
Node 2 (Master): slots 5461 - 10922   + Replica
Node 3 (Master): slots 10923 - 16383  + Replica
```

```yaml
spring:
  data:
    redis:
      cluster:
        nodes: redis1:6379,redis2:6379,redis3:6379,redis4:6379,redis5:6379,redis6:6379
        max-redirects: 3
```

| | Standalone | Sentinel | Cluster |
|--|-----------|---------|---------|
| HA | No | Yes (failover ~30s) | Yes |
| Horizontal Scale | No | No (single master writes) | Yes |
| Multi-key operations | Yes | Yes | Limited (same slot) |
| Complexity | Simple | Medium | Complex |
| Use For | Dev/Test | Production, < 1TB data | Production, huge data |

---

# 9. Interview Questions

### Basic
1. What is Redis and why use it instead of an application-level cache?
2. What are the main Redis data structures? When would you use each?
3. What is cache eviction and what are the policies?

### Intermediate
4. Explain the Cache-Aside pattern. What are its advantages and risks?
5. How does a distributed lock work in Redis?
6. What is Cache Stampede and how would you prevent it?

### Advanced
7. Explain the Redlock algorithm for distributed locking across multiple Redis nodes.
8. How would you implement rate limiting using Redis Sorted Sets?
9. When would you use Redis Sentinel vs Redis Cluster?
10. How does Redis handle data persistence (RDB vs AOF)?

### Scenario-Based
11. *Your application caches product data. A product's price is updated. How do you ensure cache consistency?*
    - `@CacheEvict` on update, or `@CachePut` with new value

12. *You're getting hammered by requests for non-existent user IDs. How do you fix it without crashing the DB?*
    - Bloom filter to reject definitely-absent IDs + cache null values with short TTL

13. *Two instances of your payment service might simultaneously try to process the same payment. How do you prevent this?*
    - Distributed lock on payment ID before processing + idempotency key check

---

## Summary — Redis Cheatsheet

```
Data Structures:
  String: JSON, counters, sessions (INCR is atomic!)
  List: queues, stacks (LPUSH/RPUSH/LPOP/RPOP)
  Set: tags, unique members (SADD/SISMEMBER/SUNION)
  Sorted Set: leaderboards, rate limiting, scheduled jobs (ZADD/ZRANK)
  Hash: object properties (HSET/HGET/HINCRBY)
  Stream: message queue (XADD/XREAD, consumer groups)

Caching Strategies:
  Cache-Aside: lazy loading, most common, app manages cache
  Write-Through: write to cache + DB synchronously
  Write-Behind: write to cache, async write to DB
  
Cache Problems & Fixes:
  Stampede: mutex lock on miss + probabilistic early expiration
  Penetration: bloom filter + cache null values
  Avalanche: TTL jitter (randomize expiry times)

Eviction Policies:
  allkeys-lru: general-purpose cache (recommended)
  volatile-lru: when you mix cache and persistent keys
  allkeys-lfu: for skewed access patterns
  
Distributed Lock:
  SET key value NX EX 30 (atomic set if not exists)
  Lua script for atomic check-and-delete on unlock
  Redisson: production library, handles renewal, fair locks

Deployment:
  Sentinel: HA with automatic failover (single master)
  Cluster: horizontal scaling (16384 hash slots sharded across masters)
```
