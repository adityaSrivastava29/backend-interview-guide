---
layout: default
title: Spring Boot
nav_order: 5
---

# Section 4: Spring Boot — Deep Dive

> Auto-configuration, Actuator, Profiles, Caching, Async, Security and more

---

## Table of Contents

1. [Auto-Configuration](#1-auto-configuration)
2. [Starter Dependencies](#2-starter-dependencies)
3. [Embedded Tomcat](#3-embedded-tomcat)
4. [Profiles & Configuration Management](#4-profiles--configuration-management)
5. [Spring Boot Actuator](#5-spring-boot-actuator)
6. [Exception Handling](#6-exception-handling)
7. [Validation](#7-validation)
8. [Caching](#8-caching)
9. [Scheduling](#9-scheduling)
10. [Async Processing](#10-async-processing)
11. [Spring Security Integration](#11-spring-security-integration)

---

## 1. Auto-Configuration

### Definition

Spring Boot's **Auto-Configuration** automatically configures Spring application based on the JARs on your classpath. If you add `spring-boot-starter-data-jpa` to your project, Spring Boot automatically configures Hibernate, JPA, and a `DataSource` — you don't write a single line of configuration.

### Mental Model

Think of Auto-Configuration like a **smart hotel room** — when you walk in, the thermostat adjusts to 22°C, the TV turns to your profile, and the coffee machine starts brewing your preferred coffee. You didn't configure anything; the room detected your presence and did the right thing.

### How Auto-Configuration Works

```mermaid
flowchart TD
    A[Spring Boot starts] --> B[Scans spring.factories / AutoConfiguration.imports]
    B --> C[Loads @EnableAutoConfiguration list]
    C --> D[For each AutoConfiguration class]
    D --> E{@ConditionalOn* conditions met?}
    E -->|Yes| F[Apply configuration - register beans]
    E -->|No| G[Skip this auto-configuration]
    F --> H[User @Configuration beans take precedence]
```

### The Key Mechanism: spring.factories / AutoConfiguration.imports

```properties
# META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
# (Spring Boot 2.7+)

org.springframework.boot.autoconfigure.data.jpa.JpaRepositoriesAutoConfiguration
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration
org.springframework.boot.autoconfigure.security.servlet.SecurityAutoConfiguration
# ... 100+ more
```

### @Conditional Annotations

```java
@Configuration
@ConditionalOnClass(DataSource.class)           // only if DataSource class exists on classpath
@ConditionalOnMissingBean(DataSource.class)     // only if user hasn't defined their own DataSource
@ConditionalOnProperty(name = "spring.datasource.url") // only if property is set
public class DataSourceAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public DataSource dataSource(DataSourceProperties properties) {
        return DataSourceBuilder.create()
            .url(properties.getUrl())
            .username(properties.getUsername())
            .password(properties.getPassword())
            .build();
    }
}
```

### Conditional Annotations Reference

| Annotation                        | Condition                                  |
| --------------------------------- | ------------------------------------------ |
| `@ConditionalOnClass`             | Class present on classpath                 |
| `@ConditionalOnMissingClass`      | Class absent from classpath                |
| `@ConditionalOnBean`              | Bean already registered                    |
| `@ConditionalOnMissingBean`       | Bean NOT registered (override opportunity) |
| `@ConditionalOnProperty`          | Property set to specific value             |
| `@ConditionalOnExpression`        | SpEL expression is true                    |
| `@ConditionalOnWebApplication`    | Is a web application                       |
| `@ConditionalOnNotWebApplication` | Is NOT a web application                   |
| `@ConditionalOnResource`          | Resource file exists                       |

### Overriding Auto-Configuration

```java
// Define your own bean → @ConditionalOnMissingBean prevents auto-config bean creation
@Configuration
public class CustomDataSourceConfig {
    @Bean  // this takes priority over auto-configured DataSource
    public DataSource dataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:postgresql://prod-db:5432/myapp");
        config.setMaximumPoolSize(50);
        config.setMinimumIdle(10);
        config.setConnectionTimeout(30000);
        return new HikariDataSource(config);
    }
}
```

### Debugging Auto-Configuration

```bash
# See all auto-configuration decisions (why applied or skipped)
java -jar app.jar --debug

# Or in application.properties:
logging.level.org.springframework.boot.autoconfigure=DEBUG

# Actuator endpoint
GET /actuator/conditions
```

---

## 2. Starter Dependencies

### What is a Starter?

A Starter is a **curated set of dependencies** you can include in your project to get all the libraries needed for a feature, with compatible versions.

| Starter                          | Includes                                  |
| -------------------------------- | ----------------------------------------- |
| `spring-boot-starter-web`        | Spring MVC, Tomcat, Jackson               |
| `spring-boot-starter-data-jpa`   | JPA, Hibernate, Spring Data JPA, HikariCP |
| `spring-boot-starter-security`   | Spring Security                           |
| `spring-boot-starter-test`       | JUnit 5, Mockito, AssertJ, MockMvc        |
| `spring-boot-starter-actuator`   | Micrometer, health endpoints              |
| `spring-boot-starter-cache`      | Spring Cache abstraction                  |
| `spring-boot-starter-data-redis` | Jedis/Lettuce, Spring Data Redis          |
| `spring-boot-starter-kafka`      | Spring Kafka                              |
| `spring-boot-starter-validation` | Hibernate Validator (JSR-380)             |

---

## 3. Embedded Tomcat

### How It Works

Spring Boot embeds Tomcat **inside your JAR**. Your app is a standalone executable — no need for external application server.

```
Traditional Deployment (WAR):
  Tomcat Server (external)
    ├── webapps/
    │   └── myapp.war  ← your code deployed here

Spring Boot Deployment (JAR):
  java -jar myapp.jar
    └── myapp.jar (executable)
        ├── BOOT-INF/classes/     ← your classes
        ├── BOOT-INF/lib/         ← all dependencies including tomcat-embed-core.jar
        └── org/springframework/boot/loader/  ← Spring Boot launcher
```

### Configuring Embedded Tomcat

```yaml
# application.yml
server:
  port: 8080
  tomcat:
    max-threads: 200 # max concurrent request threads
    min-spare-threads: 10
    accept-count: 100 # queue size when all threads busy
    connection-timeout: 20000 # 20 seconds
    max-connections: 8192
  compression:
    enabled: true
    mime-types: application/json,text/html
    min-response-size: 1024 # only compress responses > 1KB
```

```java
// Programmatic Tomcat customization
@Bean
public WebServerFactoryCustomizer<TomcatServletWebServerFactory> tomcatCustomizer() {
    return factory -> {
        factory.addConnectorCustomizers(connector -> {
            connector.setMaxPostSize(10 * 1024 * 1024); // 10MB max POST body
        });
        factory.addContextCustomizers(context -> {
            context.setSessionTimeout(30); // 30-minute session timeout
        });
    };
}
```

---

## 4. Profiles & Configuration Management

### Profiles

Profiles allow **environment-specific configuration** — different settings for dev, staging, prod.

```yaml
# application.yml (base config)
spring:
  application:
    name: order-service

server:
  port: 8080

---
# application-dev.yml
spring:
  config:
    activate:
      on-profile: dev
  datasource:
    url: jdbc:h2:mem:testdb

logging:
  level:
    com.example: DEBUG

---
# application-prod.yml
spring:
  config:
    activate:
      on-profile: prod
  datasource:
    url: jdbc:postgresql://${DB_HOST}:5432/${DB_NAME}
    username: ${DB_USER}
    password: ${DB_PASSWORD}

logging:
  level:
    com.example: WARN
```

```bash
# Activate profiles
java -jar app.jar --spring.profiles.active=prod
export SPRING_PROFILES_ACTIVE=prod
```

```java
// Profile-specific beans
@Configuration
@Profile("prod")
public class ProdSecurityConfig {
    @Bean
    public SecurityFilterChain prodSecurity(HttpSecurity http) throws Exception {
        return http
            .requiresChannel(channel -> channel.anyRequest().requiresSecure())
            .build();
    }
}

@Configuration
@Profile("dev")
public class DevSecurityConfig {
    @Bean
    public SecurityFilterChain devSecurity(HttpSecurity http) throws Exception {
        return http.csrf().disable().authorizeRequests().anyRequest().permitAll().and().build();
    }
}
```

### Configuration Priority (Highest to Lowest)

```
1. Command-line arguments: --server.port=9090
2. OS environment variables: SERVER_PORT=9090
3. application-{profile}.properties in /config
4. application-{profile}.properties in classpath
5. application.properties in /config
6. application.properties in classpath
7. @PropertySource annotations
8. Default properties
```

### Externalized Configuration Best Practices

```java
// NEVER hardcode secrets in code or application.properties
// BAD:
spring.datasource.password=mysecretpassword

// GOOD: Use environment variables
spring.datasource.password=${DB_PASSWORD}

// BETTER: Use Spring Cloud Config Server or Vault
spring:
  config:
    import: "vault://secret/myapp"

// PRODUCTION PATTERN: Use Kubernetes secrets or AWS Secrets Manager
@Value("${DB_PASSWORD}")
private String dbPassword;
```

---

## 5. Spring Boot Actuator

### Definition

Actuator provides **production-ready operational endpoints** — health checks, metrics, thread dumps, heap dumps, configuration details, and more.

### Setup

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus,loggers,threaddump,heapdump,env
        # NEVER expose 'env', 'heapdump', 'shutdown' without authentication in production!
  endpoint:
    health:
      show-details: when-authorized # don't expose db details to public
      show-components: always
  health:
    circuitbreakers:
      enabled: true
  metrics:
    export:
      prometheus:
        enabled: true
```

### Key Endpoints

| Endpoint               | Description                   | Production Use              |
| ---------------------- | ----------------------------- | --------------------------- |
| `/actuator/health`     | App health + component status | Load balancer health check  |
| `/actuator/metrics`    | JVM, HTTP, custom metrics     | Grafana dashboards          |
| `/actuator/prometheus` | Prometheus-format metrics     | Prometheus scraping         |
| `/actuator/info`       | App version, git info         | Deployment verification     |
| `/actuator/env`        | Configuration properties      | Debugging (SECURE!)         |
| `/actuator/loggers`    | Log level management          | Change log level at runtime |
| `/actuator/threaddump` | JVM thread dump               | Deadlock investigation      |
| `/actuator/heapdump`   | JVM heap dump                 | Memory leak investigation   |
| `/actuator/conditions` | Auto-config decisions         | Debugging config            |
| `/actuator/mappings`   | Request handler mappings      | API discovery               |
| `/actuator/shutdown`   | Graceful shutdown (POST)      | NEVER expose publicly!      |

### Custom Health Indicator

```java
@Component
public class ExternalApiHealthIndicator implements HealthIndicator {

    @Autowired
    private PaymentGatewayClient paymentClient;

    @Override
    public Health health() {
        try {
            ResponseEntity<String> response = paymentClient.ping();
            if (response.getStatusCode().is2xxSuccessful()) {
                return Health.up()
                    .withDetail("paymentGateway", "reachable")
                    .withDetail("latencyMs", response.getHeaders().getFirst("X-Response-Time"))
                    .build();
            }
            return Health.down()
                .withDetail("paymentGateway", "unhealthy")
                .withDetail("statusCode", response.getStatusCode())
                .build();
        } catch (Exception e) {
            return Health.down()
                .withException(e)
                .withDetail("paymentGateway", "unreachable")
                .build();
        }
    }
}
```

### Custom Metrics

```java
@Service
public class OrderService {

    private final Counter ordersPlacedCounter;
    private final Timer orderProcessingTimer;
    private final Gauge pendingOrdersGauge;

    public OrderService(MeterRegistry meterRegistry, OrderRepository orderRepo) {
        this.ordersPlacedCounter = Counter.builder("orders.placed.total")
            .description("Total orders placed")
            .tag("currency", "USD")
            .register(meterRegistry);

        this.orderProcessingTimer = Timer.builder("orders.processing.time")
            .description("Order processing duration")
            .register(meterRegistry);

        this.pendingOrdersGauge = Gauge.builder("orders.pending.count", orderRepo,
            repo -> repo.countByStatus("PENDING"))
            .description("Current pending orders")
            .register(meterRegistry);
    }

    public Order placeOrder(OrderRequest request) {
        return orderProcessingTimer.record(() -> {
            Order order = processOrder(request);
            ordersPlacedCounter.increment();
            return order;
        });
    }
}
```

---

## 6. Exception Handling

### @ControllerAdvice — Global Exception Handler

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    // Validation errors
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ApiError handleValidationErrors(MethodArgumentNotValidException ex) {
        List<String> errors = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .map(error -> error.getField() + ": " + error.getDefaultMessage())
            .collect(Collectors.toList());

        return ApiError.builder()
            .status(400)
            .message("Validation failed")
            .errors(errors)
            .timestamp(Instant.now())
            .build();
    }

    // Business logic exceptions
    @ExceptionHandler(OrderNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ApiError handleNotFound(OrderNotFoundException ex, HttpServletRequest request) {
        log.warn("Order not found: {}", ex.getOrderId());
        return ApiError.of(404, ex.getMessage(), request.getRequestURI());
    }

    // Payment failures
    @ExceptionHandler(PaymentDeclinedException.class)
    @ResponseStatus(HttpStatus.UNPROCESSABLE_ENTITY)
    public ApiError handlePaymentDeclined(PaymentDeclinedException ex) {
        log.warn("Payment declined: {}", ex.getReason());
        return ApiError.of(422, "Payment could not be processed: " + ex.getReason());
    }

    // Catch-all — unexpected errors
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ApiError handleUnexpected(Exception ex, HttpServletRequest request) {
        String correlationId = UUID.randomUUID().toString();
        log.error("Unexpected error [correlationId={}]: {}", correlationId, ex.getMessage(), ex);
        return ApiError.of(500, "An unexpected error occurred. Ref: " + correlationId);
    }
}

// Standard error response
@Builder
public class ApiError {
    private int status;
    private String message;
    private List<String> errors;
    private String path;
    private Instant timestamp;
}
```

---

## 7. Validation

```java
// Request DTO with validation constraints
@Data
public class CreateOrderRequest {

    @NotBlank(message = "User ID is required")
    private String userId;

    @NotEmpty(message = "Order must have at least one item")
    @Size(max = 50, message = "Cannot have more than 50 items")
    private List<@Valid OrderItem> items;

    @NotNull
    @DecimalMin(value = "0.01", message = "Amount must be positive")
    @DecimalMax(value = "999999.99", message = "Amount exceeds maximum")
    private BigDecimal totalAmount;

    @Pattern(regexp = "^[A-Z]{3}$", message = "Currency must be 3-letter ISO code")
    private String currency;

    @Future(message = "Delivery date must be in the future")
    private LocalDate requestedDeliveryDate;

    @Email
    private String contactEmail;
}

// Controller
@RestController
@RequestMapping("/api/orders")
@Validated  // enables method-level validation
public class OrderController {

    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(
            @Valid @RequestBody CreateOrderRequest request) { // @Valid triggers validation
        return ResponseEntity.ok(orderService.createOrder(request));
    }

    // Method-level constraint
    @GetMapping("/{orderId}")
    public OrderResponse getOrder(
            @PathVariable @NotBlank @Pattern(regexp = "^ORD-[0-9]+$") String orderId) {
        return orderService.findById(orderId);
    }
}

// Custom validator
@Constraint(validatedBy = ValidCurrencyValidator.class)
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
public @interface ValidCurrency {
    String message() default "Invalid currency code";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

public class ValidCurrencyValidator implements ConstraintValidator<ValidCurrency, String> {
    private static final Set<String> SUPPORTED = Set.of("USD", "EUR", "GBP", "INR");

    @Override
    public boolean isValid(String value, ConstraintValidatorContext ctx) {
        return value != null && SUPPORTED.contains(value.toUpperCase());
    }
}
```

---

## 8. Caching

### Spring Cache Abstraction

```java
// Enable caching
@SpringBootApplication
@EnableCaching
public class Application { ... }

// Service with caching
@Service
public class ProductService {

    // Cache result keyed by productId
    @Cacheable(value = "products", key = "#productId")
    public Product getProduct(String productId) {
        log.info("DB HIT for product: {}", productId); // only called on cache miss
        return productRepository.findById(productId).orElseThrow();
    }

    // Update cache when product is updated
    @CachePut(value = "products", key = "#product.id")
    public Product updateProduct(Product product) {
        return productRepository.save(product);
    }

    // Evict from cache when product is deleted
    @CacheEvict(value = "products", key = "#productId")
    public void deleteProduct(String productId) {
        productRepository.deleteById(productId);
    }

    // Evict all entries in the "products" cache
    @CacheEvict(value = "products", allEntries = true)
    public void refreshAllProducts() {
        // typically called by a scheduled job
    }

    // Conditional caching
    @Cacheable(value = "products", key = "#productId",
               condition = "#productId != null",
               unless = "#result == null") // don't cache null results
    public Product getProductSafe(String productId) {
        return productRepository.findById(productId).orElse(null);
    }
}
```

### Redis Cache Configuration

```java
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        RedisCacheConfiguration defaultConfig = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .serializeKeysWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new GenericJackson2JsonRedisSerializer()))
            .disableCachingNullValues();

        // Different TTL per cache
        Map<String, RedisCacheConfiguration> cacheConfigs = Map.of(
            "products", defaultConfig.entryTtl(Duration.ofHours(1)),
            "users",    defaultConfig.entryTtl(Duration.ofMinutes(30)),
            "sessions", defaultConfig.entryTtl(Duration.ofHours(8))
        );

        return RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(defaultConfig)
            .withInitialCacheConfigurations(cacheConfigs)
            .build();
    }
}
```

---

## 9. Scheduling

```java
@SpringBootApplication
@EnableScheduling
public class Application { ... }

@Component
public class ScheduledJobs {

    // Fixed rate — runs every 5 minutes, regardless of task duration
    @Scheduled(fixedRate = 5, timeUnit = TimeUnit.MINUTES)
    public void syncInventory() {
        inventoryService.syncWithWarehouse();
    }

    // Fixed delay — waits 30 seconds AFTER task completes before next run
    @Scheduled(fixedDelay = 30, timeUnit = TimeUnit.SECONDS)
    public void processRetryQueue() {
        retryService.processFailedEvents();
    }

    // Cron — every weekday at 2:00 AM
    @Scheduled(cron = "0 0 2 * * MON-FRI")
    public void generateDailyReport() {
        reportService.generateReport();
    }

    // Cron with zone
    @Scheduled(cron = "0 0 9 * * *", zone = "America/New_York")
    public void sendMorningDigest() {
        emailService.sendDailyDigest();
    }
}
```

### Cron Expression Format

```
┌──────────── second (0-59)
│ ┌────────── minute (0-59)
│ │ ┌──────── hour (0-23)
│ │ │ ┌────── day of month (1-31)
│ │ │ │ ┌──── month (1-12 or JAN-DEC)
│ │ │ │ │ ┌── day of week (0-7 or SUN-SAT, 0 and 7 = Sunday)
│ │ │ │ │ │
* * * * * *

Examples:
0 0 * * * *        → every hour
0 */15 * * * *     → every 15 minutes
0 0 9-17 * * MON-FRI → hourly 9-5 weekdays
0 0 0 1 * *        → midnight on 1st of every month
```

### Distributed Scheduling with ShedLock

```java
// Prevent multiple instances from running the same scheduled job
@Scheduled(fixedDelay = 10, timeUnit = TimeUnit.MINUTES)
@SchedulerLock(name = "syncInventory", lockAtLeastFor = "9m", lockAtMostFor = "14m")
public void syncInventory() {
    // Only one instance executes this at a time
    inventoryService.syncWithWarehouse();
}
```

---

## 10. Async Processing

```java
@SpringBootApplication
@EnableAsync
public class Application { ... }

// Configure async executor
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(500);
        executor.setThreadNamePrefix("async-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (ex, method, params) -> {
            log.error("Async method {} failed: {}", method.getName(), ex.getMessage(), ex);
            alertingService.sendAlert("Async failure in " + method.getName());
        };
    }
}

@Service
public class NotificationService {

    // @Async — returns immediately, runs in background thread
    @Async
    public void sendEmailAsync(String recipient, String subject, String body) {
        // runs in thread pool, does not block caller
        emailClient.send(recipient, subject, body);
    }

    // @Async with Future return
    @Async
    public CompletableFuture<NotificationResult> sendPushAsync(String userId, String message) {
        try {
            PushResult result = pushClient.send(userId, message);
            return CompletableFuture.completedFuture(NotificationResult.success(result));
        } catch (Exception e) {
            return CompletableFuture.failedFuture(e);
        }
    }
}

// Controller using async
@RestController
public class OrderController {

    @PostMapping("/orders")
    public ResponseEntity<OrderResponse> placeOrder(@RequestBody OrderRequest request) {
        Order order = orderService.createOrder(request);

        // Fire-and-forget: send notifications asynchronously
        notificationService.sendEmailAsync(request.getEmail(), "Order Confirmed", buildEmailBody(order));
        notificationService.sendPushAsync(request.getUserId(), "Your order #" + order.getId() + " is confirmed!");

        // Return immediately without waiting for notifications
        return ResponseEntity.ok(OrderResponse.from(order));
    }
}
```

---

## 11. Spring Security Integration

> Full Security coverage is in Section 13. This section covers Spring Boot integration.

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf.disable()) // disabled for stateless REST APIs
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**", "/actuator/health").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers(HttpMethod.GET, "/api/products/**").hasAnyRole("USER", "ADMIN")
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint(jwtAuthEntryPoint)
                .accessDeniedHandler(jwtAccessDeniedHandler)
            )
            .build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12); // cost factor 12
    }

    @Bean
    public AuthenticationManager authManager(AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }
}
```

---

### Production Scenarios

#### Scenario 1: Application OOMing After Deploy

**Problem:** Application running fine for 2 hours, then crashes with OOM.

**Root Cause:** `@Cacheable` on a method returning a `List<Order>` without TTL — cache grows unbounded.

**Fix:**

```yaml
spring:
  cache:
    redis:
      time-to-live: 600000 # 10 minutes in ms
```

**Prevention:** Always set TTL on cache entries. Use `@CacheEvict` on write operations. Monitor cache hit/miss ratio via Actuator metrics.

---

#### Scenario 2: @Transactional Not Working

**Problem:** Data is not being rolled back after an exception.

**Root Cause:** `@Transactional` on a method that catches the exception and doesn't rethrow it. Or method called via self-invocation.

**Fix:**

```java
// WRONG
@Transactional
public void processOrder(Order order) {
    try {
        chargePayment(order);
        updateInventory(order); // fails
    } catch (InventoryException e) {
        log.error("Failed", e); // swallows exception → NO rollback!
    }
}

// RIGHT
@Transactional
public void processOrder(Order order) {
    chargePayment(order);
    updateInventory(order); // exception propagates → rollback
}
```

---

### Interview Questions

#### Basic

1. What is Auto-Configuration in Spring Boot?
2. How do you create a custom health indicator?
3. What is the difference between `@Scheduled(fixedRate)` and `@Scheduled(fixedDelay)`?

#### Intermediate

4. How does `@ConditionalOnMissingBean` enable overriding auto-configured beans?
5. How would you configure different database settings for dev and prod profiles?
6. What is the difference between `@Async` and `CompletableFuture.supplyAsync`?

#### Advanced

7. How does Spring Boot's embedded Tomcat handle graceful shutdown?
8. Explain how `@Cacheable` works with Redis — trace from annotation to Redis call.
9. How would you implement rate limiting using Spring Boot and Redis?
10. How do you prevent distributed scheduling jobs from running on multiple nodes?

---

### Summary — Spring Boot Cheatsheet

```
Auto-Configuration:
  - Scans AutoConfiguration.imports, applies @Conditional* checks
  - Override by defining your own bean (@ConditionalOnMissingBean skips auto-config)
  - Debug: --debug flag or /actuator/conditions

Profiles: dev | staging | prod
  - application-{profile}.yml per environment
  - Activate: SPRING_PROFILES_ACTIVE=prod
  - Priority: CLI args > env vars > profile props > base props

Actuator Key Endpoints:
  /health → load balancer probe
  /metrics → Grafana dashboards
  /prometheus → Prometheus scraping
  /loggers → change log level at runtime (no redeploy!)
  /threaddump → deadlock investigation

Exception Handling:
  @RestControllerAdvice + @ExceptionHandler → global handler
  Always return consistent error schema with correlationId

Caching:
  @Cacheable → read cache
  @CachePut → write cache on update
  @CacheEvict → invalidate on delete
  Always set TTL to prevent unbounded growth

Async:
  @EnableAsync + @Async → runs in separate thread pool
  Configure ThreadPoolTaskExecutor with bounded queue
  Handle async exceptions via AsyncUncaughtExceptionHandler

Scheduling:
  @EnableScheduling + @Scheduled
  Use ShedLock for distributed environments (prevents duplicate execution)
```
