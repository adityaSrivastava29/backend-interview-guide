---
layout: default
title: Spring Cloud
nav_order: 9
---

# Section 8: Spring Cloud — Service Mesh for Java

> Eureka, Config Server, API Gateway, OpenFeign, Resilience4j — production examples

---

## Table of Contents

1. [Eureka — Service Discovery](#1-eureka--service-discovery)
2. [Spring Cloud Config Server](#2-spring-cloud-config-server)
3. [Spring Cloud Gateway](#3-spring-cloud-gateway)
4. [OpenFeign — Declarative HTTP Client](#4-openfeign--declarative-http-client)
5. [Resilience4j — Fault Tolerance](#5-resilience4j--fault-tolerance)
6. [Spring Cloud Sleuth & Distributed Tracing](#6-distributed-tracing)

---

# 1. Eureka — Service Discovery

## Setup: Eureka Server

```xml
<!-- pom.xml: Eureka Server -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

```java
@SpringBootApplication
@EnableEurekaServer
public class ServiceRegistryApplication { }
```

```yaml
# application.yml — Eureka Server
server:
  port: 8761

eureka:
  client:
    register-with-eureka: false   # server doesn't register with itself
    fetch-registry: false
  server:
    enable-self-preservation: false   # disable in dev (re-enable in prod!)
    eviction-interval-timer-in-ms: 10000
```

## Setup: Eureka Client (Microservice)

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

```yaml
# application.yml — microservice
spring:
  application:
    name: order-service   # THIS is the service name registered in Eureka

eureka:
  client:
    service-url:
      defaultZone: http://eureka-server:8761/eureka
    registry-fetch-interval-seconds: 5
  instance:
    prefer-ip-address: true
    instance-id: ${spring.application.name}:${server.port}
    lease-renewal-interval-in-seconds: 10    # heartbeat interval
    lease-expiration-duration-in-seconds: 30  # removed if no heartbeat
    metadata-map:
      version: "2.1.0"
      environment: "production"
```

## Eureka Self-Preservation

**Problem:** If Eureka server loses heartbeats from 85% of clients (e.g., network issue), it enters **self-preservation mode** and stops evicting instances — even unhealthy ones.

**In Production:** Keep self-preservation enabled. In development, disable it for faster testing.

---

# 2. Spring Cloud Config Server

## What It Does

Centralized configuration management — all microservices pull their config from one place (Git repo, Vault, filesystem).

```
Git Repository (centralized config)
├── application.yml              (shared across all services)
├── order-service.yml            (order-service specific)
├── order-service-prod.yml       (order-service production overrides)
├── payment-service.yml
└── payment-service-prod.yml
```

## Config Server Setup

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
```

```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication { }
```

```yaml
# Config Server application.yml
server:
  port: 8888

spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/mycompany/config-repo
          default-label: main
          search-paths: "{application}"  # look in subdirectory named after service
          clone-on-start: true
          timeout: 10
          
          # For private repos
          username: ${GIT_USER}
          password: ${GIT_TOKEN}
          
          # Or with SSH key
          private-key: ${GIT_SSH_KEY}

# Encrypt sensitive properties
encrypt:
  key: ${ENCRYPTION_KEY}  # symmetric key for {cipher} values
```

## Config Client Setup

```yaml
# bootstrap.yml (loaded BEFORE application.yml)
spring:
  application:
    name: order-service
  cloud:
    config:
      uri: http://config-server:8888
      fail-fast: true       # fail startup if config server unreachable
      retry:
        max-attempts: 6
        initial-interval: 1000

# OR in application.yml with Spring Boot 2.4+:
spring:
  config:
    import: "optional:configserver:http://config-server:8888"
```

## Encrypting Sensitive Config

```bash
# Encrypt a value via Config Server API
curl http://config-server:8888/encrypt -d "mySecretPassword"
# Returns: 682bc583f4641835fa2db009355293665d2647dade3375c0ee201de2a49f7bda

# Store in Git config file:
# spring.datasource.password: '{cipher}682bc583f4641835fa2db009355293665d2647dade3375c0ee201de2a49f7bda'
```

## Dynamic Config Refresh

```java
@RestController
@RefreshScope  // reloads bean when /actuator/refresh called
public class FeatureFlagController {
    
    @Value("${feature.newCheckout.enabled:false}")
    private boolean newCheckoutEnabled;
    
    @GetMapping("/features/checkout")
    public boolean isNewCheckoutEnabled() {
        return newCheckoutEnabled;
    }
}
```

```bash
# Trigger refresh across all instances via Spring Cloud Bus + Kafka/RabbitMQ
POST /actuator/busrefresh

# Or refresh single instance
POST /actuator/refresh
```

---

# 3. Spring Cloud Gateway

## Advanced Configuration

```java
@Configuration
public class GatewayRoutingConfig {
    
    @Bean
    public RouteLocator gatewayRoutes(RouteLocatorBuilder builder,
                                      JwtAuthFilter jwtFilter,
                                      RequestLoggingFilter loggingFilter) {
        return builder.routes()
            
            // Order Service route with full configuration
            .route("order-service", r -> r
                .path("/api/v1/orders/**")
                .and()
                .method(HttpMethod.GET, HttpMethod.POST, HttpMethod.PUT)
                .filters(f -> f
                    .filter(jwtFilter.apply(new JwtAuthFilter.Config()))
                    .filter(loggingFilter.apply(new RequestLoggingFilter.Config()))
                    .addRequestHeader("X-Forwarded-Service", "api-gateway")
                    .addRequestHeader("X-Request-ID", UUID.randomUUID().toString())
                    .removeRequestHeader("Cookie")
                    .rewritePath("/api/v1/(?<segment>.*)", "/api/${segment}")
                    .circuitBreaker(c -> c
                        .setName("orderCB")
                        .setFallbackUri("forward:/fallback/service-unavailable"))
                    .retry(config -> config
                        .setRetries(3)
                        .setStatuses(HttpStatus.INTERNAL_SERVER_ERROR)
                        .setMethods(HttpMethod.GET)
                        .setBackoff(Duration.ofMillis(100), Duration.ofSeconds(1), 2, true))
                    .requestRateLimiter(config -> config
                        .setRateLimiter(redisRateLimiter())
                        .setKeyResolver(userKeyResolver()))
                )
                .uri("lb://order-service"))
            
            // Public endpoint — no auth
            .route("product-catalog-public", r -> r
                .path("/api/v1/products/**")
                .and()
                .method(HttpMethod.GET)
                .filters(f -> f
                    .cache(Duration.ofMinutes(5))  // cache GET responses
                )
                .uri("lb://product-service"))
            
            .build();
    }
    
    // Rate limiter: 100 requests/second per user
    @Bean
    public RedisRateLimiter redisRateLimiter() {
        return new RedisRateLimiter(100, 200, 1);  // replenishRate, burstCapacity, requestedTokens
    }
    
    // Rate limit key: extract user ID from JWT
    @Bean
    public KeyResolver userKeyResolver() {
        return exchange -> {
            String userId = exchange.getRequest().getHeaders().getFirst("X-User-ID");
            return Mono.just(userId != null ? userId : "anonymous");
        };
    }
}
```

## Custom Global Filter

```java
@Component
public class RequestLoggingGlobalFilter implements GlobalFilter, Ordered {
    
    private static final Logger log = LoggerFactory.getLogger(RequestLoggingGlobalFilter.class);
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        long startTime = System.currentTimeMillis();
        String requestId = UUID.randomUUID().toString();
        
        // Mutate request: add correlation ID
        ServerHttpRequest mutatedRequest = exchange.getRequest().mutate()
            .header("X-Correlation-ID", requestId)
            .build();
        
        return chain.filter(exchange.mutate().request(mutatedRequest).build())
            .doOnSuccess(v -> {
                long duration = System.currentTimeMillis() - startTime;
                log.info("[{}] {} {} → {} ({}ms)",
                    requestId,
                    exchange.getRequest().getMethod(),
                    exchange.getRequest().getURI().getPath(),
                    exchange.getResponse().getStatusCode(),
                    duration);
            })
            .doOnError(e -> log.error("[{}] Request failed: {}", requestId, e.getMessage()));
    }
    
    @Override
    public int getOrder() { return -1; } // run before other filters
}
```

---

# 4. OpenFeign — Declarative HTTP Client

## Definition

**OpenFeign** lets you write HTTP client code as Java interfaces with annotations — no `RestTemplate` or `WebClient` boilerplate.

## Setup

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

```java
@SpringBootApplication
@EnableFeignClients
public class OrderServiceApplication { }
```

## Feign Client Interface

```java
@FeignClient(
    name = "inventory-service",             // Eureka service name
    url = "${services.inventory.url}",      // override for testing
    configuration = InventoryClientConfig.class,
    fallback = InventoryClientFallback.class
)
public interface InventoryClient {
    
    @GetMapping("/api/inventory/{productId}")
    InventoryStatus checkStock(@PathVariable("productId") String productId);
    
    @PostMapping("/api/inventory/reserve")
    ReservationResult reserve(@RequestBody ReservationRequest request);
    
    @DeleteMapping("/api/inventory/reserve/{reservationId}")
    void cancelReservation(@PathVariable String reservationId);
    
    @GetMapping("/api/inventory/bulk")
    List<InventoryStatus> checkBulkStock(@RequestParam("ids") List<String> productIds);
}
```

## Feign Configuration

```java
@Configuration
public class InventoryClientConfig {
    
    // Custom request interceptor (add auth header to all requests)
    @Bean
    public RequestInterceptor authInterceptor() {
        return template -> {
            // Propagate JWT token from incoming request to outgoing
            String token = SecurityContextHolder.getContext()
                .getAuthentication().getCredentials().toString();
            template.header("Authorization", "Bearer " + token);
            template.header("X-Service-Name", "order-service");
        };
    }
    
    // Custom error decoder
    @Bean
    public ErrorDecoder errorDecoder() {
        return (methodKey, response) -> {
            if (response.status() == 404) {
                return new ProductNotFoundException("Product not found");
            }
            if (response.status() == 409) {
                return new InsufficientStockException("Insufficient stock");
            }
            if (response.status() >= 500) {
                return new RetryableException(
                    response.status(),
                    "Server error from inventory service",
                    response.request().httpMethod(),
                    null, response.request()
                );
            }
            return FeignException.errorStatus(methodKey, response);
        };
    }
    
    // Custom encoder/decoder
    @Bean
    public Encoder feignEncoder(ObjectMapper objectMapper) {
        return new JacksonEncoder(objectMapper);
    }
    
    // Logging
    @Bean
    public Logger.Level feignLoggerLevel() {
        return Logger.Level.FULL; // NONE, BASIC, HEADERS, FULL
    }
    
    // Timeout
    @Bean
    public Request.Options requestOptions() {
        return new Request.Options(
            Duration.ofSeconds(2),   // connect timeout
            Duration.ofSeconds(5),   // read timeout
            true                     // follow redirects
        );
    }
}
```

## Feign + Resilience4j

```java
@FeignClient(
    name = "payment-service",
    fallbackFactory = PaymentClientFallbackFactory.class  // provides reason for fallback
)
public interface PaymentClient {
    @PostMapping("/api/payments/charge")
    PaymentResult charge(@RequestBody ChargeRequest request);
}

@Component
public class PaymentClientFallbackFactory implements FallbackFactory<PaymentClient> {
    
    @Override
    public PaymentClient create(Throwable cause) {
        log.warn("Payment service fallback triggered: {}", cause.getMessage());
        return request -> {
            if (cause instanceof CircuitBreakerOpenException) {
                return PaymentResult.unavailable("Payment service circuit open");
            }
            if (cause instanceof FeignException.ServiceUnavailable) {
                return PaymentResult.queued("Payment queued for retry");
            }
            return PaymentResult.failed("Payment temporarily unavailable");
        };
    }
}
```

## OpenFeign vs RestTemplate vs WebClient

| | RestTemplate | WebClient | OpenFeign |
|--|-------------|-----------|-----------|
| API Style | Imperative, verbose | Reactive, fluent | Declarative interface |
| Blocking | Yes | Non-blocking | Yes |
| Load Balancing | Manual | Manual | Auto (via Eureka) |
| Circuit Breaking | Manual | Manual | Auto (Resilience4j) |
| Retry | Manual | Manual | Auto |
| Code Volume | High | Medium | Minimal |
| Spring Boot 3 | Deprecated | Recommended | Recommended |
| Best For | Legacy | High concurrency | Service-to-service |

---

# 5. Resilience4j — Fault Tolerance

## Complete Configuration

```yaml
# application.yml
resilience4j:
  
  circuitbreaker:
    instances:
      paymentGateway:
        register-health-indicator: true
        sliding-window-type: COUNT_BASED
        sliding-window-size: 10
        failure-rate-threshold: 50
        wait-duration-in-open-state: 30s
        permitted-number-of-calls-in-half-open-state: 3
        automatic-transition-from-open-to-half-open-enabled: true
        record-exceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
          - feign.FeignException.ServiceUnavailable
        ignore-exceptions:
          - com.example.exception.BusinessException
  
  retry:
    instances:
      inventoryService:
        max-attempts: 3
        wait-duration: 500ms
        retry-exceptions:
          - java.io.IOException
        ignore-exceptions:
          - com.example.exception.NotFoundException
        exponential-backoff-multiplier: 2
        enable-exponential-backoff: true
  
  ratelimiter:
    instances:
      smsService:
        limit-for-period: 100          # max 100 calls per refresh period
        limit-refresh-period: 1s       # reset every second
        timeout-duration: 0s           # fail immediately if rate exceeded
  
  bulkhead:
    instances:
      paymentGateway:
        max-concurrent-calls: 10        # max 10 concurrent calls
        max-wait-duration: 100ms        # wait up to 100ms for permit
  
  timelimiter:
    instances:
      paymentGateway:
        timeout-duration: 2s
        cancel-running-future: true
```

```java
// Stacking multiple resilience annotations (evaluated inner-to-outer)
@CircuitBreaker(name = "paymentGateway", fallbackMethod = "paymentFallback")
@TimeLimiter(name = "paymentGateway")          // timeout applied first
@Bulkhead(name = "paymentGateway")             // then bulkhead
@Retry(name = "paymentGateway")                // then retry
public CompletableFuture<PaymentResult> chargeCard(ChargeRequest request) {
    return CompletableFuture.supplyAsync(() -> gateway.charge(request));
}

public CompletableFuture<PaymentResult> paymentFallback(ChargeRequest request, Exception e) {
    return CompletableFuture.completedFuture(PaymentResult.queued("Payment queued"));
}
```

---

# 6. Distributed Tracing

## Problem

```
User request spans 5 services:
  API Gateway → Order Service → Inventory Service → Payment Service → Notification Service
  
Which service is causing the 2-second latency? Without tracing, impossible to tell.
```

## Solution: Micrometer Tracing (Spring Boot 3) / Sleuth (Spring Boot 2)

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>
<dependency>
    <groupId>io.zipkin.reporter2</groupId>
    <artifactId>zipkin-reporter-brave</artifactId>
</dependency>
```

```yaml
management:
  tracing:
    sampling:
      probability: 0.1   # trace 10% of requests (100% in dev)
  zipkin:
    tracing:
      endpoint: http://zipkin:9411/api/v2/spans
```

## How Tracing Works

```
Request → API Gateway
  TraceId: abc-123  SpanId: span-1
  
  → Order Service (same TraceId, new SpanId)
  TraceId: abc-123  SpanId: span-2  ParentSpanId: span-1
  
    → Inventory Service
    TraceId: abc-123  SpanId: span-3  ParentSpanId: span-2
    
    → Payment Service
    TraceId: abc-123  SpanId: span-4  ParentSpanId: span-2

All spans with TraceId: abc-123 show the complete request journey
Zipkin UI shows: API Gateway(50ms) → Order(800ms) → Inventory(100ms) + Payment(1200ms)
→ Payment is the bottleneck!
```

```java
// Custom span for business operations
@Service
public class OrderService {
    
    @Autowired
    private Tracer tracer;
    
    public Order placeOrder(OrderRequest request) {
        Span span = tracer.nextSpan().name("place-order").start();
        
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            span.tag("order.user_id", request.getUserId());
            span.tag("order.amount", request.getTotalAmount().toString());
            
            Order order = processOrder(request);
            
            span.tag("order.id", order.getId().toString());
            return order;
        } catch (Exception e) {
            span.error(e);
            throw e;
        } finally {
            span.end();
        }
    }
}
```

---

## Production Scenarios

### Scenario: Config Server Down on Startup

**Problem:** All microservices fail to start because Config Server is unavailable.

**Root Cause:** Default behavior is to fail fast when Config Server unreachable.

**Solution:**
```yaml
spring:
  cloud:
    config:
      fail-fast: false   # don't crash if config server down
      retry:
        max-attempts: 6
        initial-interval: 2000
        max-interval: 30000
      # Use local fallback config
      import: "optional:configserver:http://config-server:8888"
```

---

## Interview Questions

### Basic
1. What is Eureka and what problem does it solve?
2. What is the difference between Eureka server and Eureka client?
3. What is OpenFeign and how is it different from RestTemplate?

### Intermediate
4. How does Spring Cloud Config Server handle encrypted properties?
5. How would you implement circuit breaking in OpenFeign?
6. What is Spring Cloud Gateway and how does it differ from Zuul?

### Advanced
7. How does distributed tracing work? What is TraceId vs SpanId?
8. How would you implement a custom Feign error decoder to handle retryable vs non-retryable errors?
9. How do you implement dynamic config refresh without restarting services?
10. Design a Spring Cloud setup for a 10-service e-commerce platform.

---

## Summary — Spring Cloud Cheatsheet

```
Eureka: Service Registry
  Server: @EnableEurekaServer, port 8761
  Client: spring.application.name + eureka.client.service-url
  Heartbeat: 30s, expiry: 90s
  
Config Server: Centralized config from Git/Vault
  Client: bootstrap.yml + spring.config.import
  Refresh: @RefreshScope + /actuator/refresh or busrefresh
  Secrets: {cipher} prefix + encrypt.key

Gateway (Spring Cloud Gateway):
  Routes: path, method, header predicates
  Filters: auth, rate limit, circuit breaker, retry, rewrite
  Rate Limiting: Redis token bucket
  
OpenFeign:
  @FeignClient(name="service-name") + @EnableFeignClients
  Config: RequestInterceptor (auth), ErrorDecoder, Options (timeout)
  Resilience: FallbackFactory + Resilience4j annotations
  
Resilience4j:
  @CircuitBreaker → fail fast on error threshold
  @Retry → exponential backoff with jitter
  @Bulkhead → thread pool isolation
  @RateLimiter → calls per second limit
  @TimeLimiter → max execution time

Tracing: TraceId (request) + SpanId (service hop)
  Propagated via B3 headers across service calls
  Exported to Zipkin/Jaeger for visualization
```
