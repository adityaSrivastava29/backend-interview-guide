---
layout: default
title: DevOps
nav_order: 13
---

# Section 12: DevOps — Docker, Kubernetes, CI/CD, Monitoring

> Container orchestration, pipelines, and observability for Java backend engineers

---

## Table of Contents

1. [Docker](#1-docker)
2. [Kubernetes](#2-kubernetes)
3. [CI/CD Pipelines](#3-cicd-pipelines)
4. [Monitoring & Observability](#4-monitoring--observability)
5. [Deployment Strategies](#5-deployment-strategies)
6. [Interview Questions](#6-interview-questions)

---

## 1. Docker

### Dockerfile Best Practices

#### Bad Dockerfile (Anti-patterns)

```dockerfile
# BAD: large image, no layer caching, runs as root
FROM ubuntu:latest
RUN apt-get update && apt-get install -y openjdk-17-jdk
COPY . /app
WORKDIR /app
RUN mvn clean package
CMD ["java", "-jar", "target/app.jar"]
```

Problems:

- `ubuntu + JDK` = huge base image (~600 MB)
- `COPY . /app` before build invalidates ALL layers on any file change
- Runs as root (security risk)
- No JVM tuning flags

#### Good Dockerfile (Multi-stage)

```dockerfile
# Stage 1: Build
FROM maven:3.9-eclipse-temurin-17 AS builder
WORKDIR /build

# Copy only POM first (to cache dependency downloads separately from code)
COPY pom.xml .
RUN mvn dependency:go-offline -q

# Now copy source (this layer invalidates only when source changes)
COPY src ./src
RUN mvn clean package -DskipTests -q

# Stage 2: Extract layers (Spring Boot layertools)
FROM eclipse-temurin:17-jre AS extractor
WORKDIR /extracted
COPY --from=builder /build/target/*.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

# Stage 3: Runtime — minimal image
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app

# Non-root user (security best practice)
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# Copy Spring Boot layers in order of change frequency
# (least → most frequently changed = better layer caching on updates)
COPY --from=extractor /extracted/dependencies/ ./
COPY --from=extractor /extracted/spring-boot-loader/ ./
COPY --from=extractor /extracted/snapshot-dependencies/ ./
COPY --from=extractor /extracted/application/ ./

EXPOSE 8080

# JVM tuning for container environment
ENTRYPOINT ["java", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \
  "-XX:+UseG1GC", \
  "-XX:+OptimizeStringConcat", \
  "-Djava.security.egd=file:/dev/./urandom", \
  "org.springframework.boot.loader.JarLauncher"]
```

### Key Dockerfile Tips

| Tip                                 | Why                                                  |
| ----------------------------------- | ---------------------------------------------------- |
| Use multi-stage builds              | Separate build tools from runtime — smaller image    |
| Use `eclipse-temurin:17-jre-alpine` | ~200MB vs ~600MB for JDK ubuntu                      |
| COPY POM first, then src            | Cache dependency downloads separately                |
| Non-root USER                       | Security: container escape has fewer privileges      |
| `-XX:+UseContainerSupport`          | JVM reads cgroup limits (not host memory)            |
| `-XX:MaxRAMPercentage=75.0`         | JVM heap = 75% of container memory limit             |
| `.dockerignore`                     | Exclude target/, .git/, README.md from build context |

```
# .dockerignore
target/
.git/
*.md
.idea/
*.log
docker-compose*.yml
```

---

## 2. Kubernetes

### Core Objects

#### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production
  labels:
    app: order-service
    version: "2.1.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1 # create 1 extra pod before killing old
      maxUnavailable: 0 # never have fewer than 3 pods
  template:
    metadata:
      labels:
        app: order-service
        version: "2.1.0"
    spec:
      containers:
        - name: order-service
          image: myregistry/order-service:2.1.0
          ports:
            - containerPort: 8080

          # Resource limits (CRITICAL: always set these!)
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m" # 0.25 cores
            limits:
              memory: "512Mi"
              cpu: "500m" # 0.5 cores

          # Environment from ConfigMap and Secrets
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "production"
            - name: DB_HOST
              valueFrom:
                configMapKeyRef:
                  name: order-service-config
                  key: db.host
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: order-service-secrets
                  key: db.password

          # Health probes
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
            # If fails: pod RESTARTED (app is dead)

          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 5
            failureThreshold: 3
            # If fails: pod REMOVED from service endpoints (app not ready)

          startupProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 30 # allow 150s for startup
            # Disables liveness/readiness until this passes (handles slow startup)
```

#### Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: production
spec:
  selector:
    app: order-service
  ports:
    - port: 80 # service port (external)
      targetPort: 8080 # container port
  type: ClusterIP # ClusterIP: internal only, NodePort: external, LoadBalancer: cloud LB
```

#### ConfigMap & Secret

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-service-config
data:
  db.host: "postgres.production.svc.cluster.local"
  db.port: "5432"
  kafka.bootstrap-servers: "kafka:9092"

---
apiVersion: v1
kind: Secret
metadata:
  name: order-service-secrets
type: Opaque
data:
  # base64 encoded values
  db.password: c3VwZXJzZWNyZXQ= # echo -n "supersecret" | base64
  jwt.secret: bXlqd3RzZWNyZXQ=
```

#### HorizontalPodAutoscaler (HPA)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70 # scale out when avg CPU > 70%
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0 # scale up immediately
    scaleDown:
      stabilizationWindowSeconds: 300 # wait 5 min before scaling down
```

---

## 3. CI/CD Pipelines

### GitHub Actions — Spring Boot Pipeline

```yaml
# .github/workflows/ci-cd.yml
name: Java CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: "17"
          distribution: "temurin"
          cache: maven

      - name: Run Tests
        run: mvn test
        env:
          SPRING_DATASOURCE_URL: jdbc:postgresql://localhost:5432/testdb

      - name: Code Coverage
        run: mvn jacoco:report

      - name: Upload Coverage to Codecov
        uses: codecov/codecov-action@v3

      - name: SonarCloud Analysis
        run: mvn sonar:sonar
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}

  build-and-push:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}

    steps:
      - uses: actions/checkout@v4

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=,suffix=,format=short
            type=raw,value=latest

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    environment: production

    steps:
      - uses: actions/checkout@v4

      - name: Set up kubectl
        uses: azure/setup-kubectl@v3

      - name: Configure kubectl
        run: echo "${{ secrets.KUBE_CONFIG }}" | base64 -d > ~/.kube/config

      - name: Deploy to Kubernetes
        run: |
          kubectl set image deployment/order-service \
            order-service=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} \
            -n production

          kubectl rollout status deployment/order-service -n production --timeout=5m

      - name: Rollback on failure
        if: failure()
        run: kubectl rollout undo deployment/order-service -n production
```

---

## 4. Monitoring & Observability

### The Three Pillars

```
Metrics: Aggregated numbers over time (e.g., "HTTP 500 errors = 42 in last 5 min")
         → Prometheus + Grafana

Logs: Discrete event records (e.g., "2024-01-01 ERROR: NullPointerException at...")
      → ELK Stack (Elasticsearch, Logstash, Kibana) or Datadog

Traces: Request journey across services (e.g., "Order API → Inventory → Payment")
        → Zipkin, Jaeger, Datadog APM
```

### Prometheus + Spring Boot Actuator

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,metrics
  metrics:
    distribution:
      percentiles-histogram:
        http.server.requests: true # enable histogram for latency percentiles
      slo:
        http.server.requests: 50ms,100ms,200ms,500ms,1s # SLO buckets
```

### Custom Metrics

```java
@Service
public class OrderMetrics {

    private final Counter ordersCreated;
    private final Counter ordersFailed;
    private final Timer orderProcessingTime;
    private final Gauge activeOrders;
    private final AtomicInteger activeOrdersCount = new AtomicInteger(0);

    public OrderMetrics(MeterRegistry registry) {
        this.ordersCreated = Counter.builder("orders.created.total")
            .description("Total orders successfully created")
            .tag("service", "order-service")
            .register(registry);

        this.ordersFailed = Counter.builder("orders.failed.total")
            .description("Total orders that failed")
            .register(registry);

        this.orderProcessingTime = Timer.builder("orders.processing.duration")
            .description("Time to process an order end-to-end")
            .publishPercentileHistogram()
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);

        this.activeOrders = Gauge.builder("orders.active", activeOrdersCount, AtomicInteger::get)
            .register(registry);
    }

    public Order processOrder(OrderRequest request) {
        activeOrdersCount.incrementAndGet();
        return orderProcessingTime.record(() -> {
            try {
                Order order = doProcessOrder(request);
                ordersCreated.increment();
                return order;
            } catch (Exception e) {
                ordersFailed.increment();
                throw e;
            } finally {
                activeOrdersCount.decrementAndGet();
            }
        });
    }
}
```

### Grafana Dashboard Queries (PromQL)

```promql
# Request rate (req/sec)
rate(http_server_requests_seconds_count[5m])

# Error rate percentage
rate(http_server_requests_seconds_count{status=~"5.."}[5m]) /
rate(http_server_requests_seconds_count[5m]) * 100

# P99 latency
histogram_quantile(0.99, rate(http_server_requests_seconds_bucket[5m]))

# JVM heap usage
jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"} * 100

# Kafka consumer lag
kafka_consumer_fetch_manager_records_lag_max
```

### Alerting Rules

```yaml
# prometheus/alerts.yml
groups:
  - name: spring-boot-alerts
    rules:
      - alert: HighErrorRate
        expr: |
          rate(http_server_requests_seconds_count{status=~"5.."}[5m]) /
          rate(http_server_requests_seconds_count[5m]) > 0.05
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "High error rate on {{ $labels.application }}"
          description: "Error rate is {{ $value | humanizePercentage }}"

      - alert: HighLatency
        expr: |
          histogram_quantile(0.99,
            rate(http_server_requests_seconds_bucket[5m])) > 2.0
        for: 5m
        annotations:
          summary: "P99 latency above 2s for {{ $labels.uri }}"

      - alert: JVMHeapHighUsage
        expr: |
          jvm_memory_used_bytes{area="heap"} / 
          jvm_memory_max_bytes{area="heap"} > 0.85
        for: 10m
        annotations:
          summary: "JVM heap > 85% on {{ $labels.instance }}"
```

### Structured Logging

```java
// Log with correlation ID and structured fields
@Component
public class OrderController {

    private static final Logger log = LoggerFactory.getLogger(OrderController.class);

    @PostMapping("/orders")
    public ResponseEntity<Order> createOrder(
            @RequestBody OrderRequest request,
            @RequestHeader("X-Correlation-ID") String correlationId) {

        // MDC: Mapped Diagnostic Context — adds fields to all log statements in this thread
        MDC.put("correlationId", correlationId);
        MDC.put("userId", request.getUserId().toString());

        try {
            log.info("Creating order for user={} items={}", request.getUserId(), request.getItemCount());
            Order order = orderService.create(request);
            log.info("Order created orderId={} totalAmount={}", order.getId(), order.getTotalAmount());
            return ResponseEntity.ok(order);
        } catch (Exception e) {
            log.error("Order creation failed reason={}", e.getMessage(), e);
            throw e;
        } finally {
            MDC.clear();
        }
    }
}

// Logback configuration — JSON format for ELK
// logback-spring.xml
```

```xml
<!-- logback-spring.xml -->
<configuration>
  <springProfile name="production">
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
      <encoder class="net.logstash.logback.encoder.LogstashEncoder">
        <!-- JSON output: {"level":"INFO","message":"...","correlationId":"...","@timestamp":"..."} -->
      </encoder>
    </appender>
  </springProfile>

  <springProfile name="local">
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
      <encoder>
        <pattern>%d{HH:mm:ss} %-5level [%X{correlationId}] %logger{36} - %msg%n</pattern>
      </encoder>
    </appender>
  </springProfile>

  <root level="INFO">
    <appender-ref ref="STDOUT"/>
  </root>
</configuration>
```

---

## 5. Deployment Strategies

### Rolling Update (Default in Kubernetes)

```
Old: [v1] [v1] [v1] [v1]
Step 1: [v2] [v1] [v1] [v1]   (1 new pod, 3 old)
Step 2: [v2] [v2] [v1] [v1]
Step 3: [v2] [v2] [v2] [v1]
Step 4: [v2] [v2] [v2] [v2]   all new

+ No downtime
+ Gradual rollout
- Both versions live simultaneously (DB must be backward-compatible!)
- Slow rollback (need to roll back all pods)
```

### Blue-Green Deployment

```
Blue (v1): [v1] [v1] [v1] ← traffic (100%)
Green (v2): [v2] [v2] [v2] ← no traffic (staging)

Switch: LB routes 100% traffic from Blue → Green
- Instant rollback: switch LB back to Blue
- Double infrastructure cost during switch
- No mixed-version traffic
```

```bash
# Kubernetes blue-green switch (change service selector)
kubectl patch service order-service -p '{"spec":{"selector":{"version":"v2"}}}'

# Rollback
kubectl patch service order-service -p '{"spec":{"selector":{"version":"v1"}}}'
```

### Canary Deployment

```
v1: [v1] [v1] [v1] [v1] ← 95% traffic
v2: [v2]                 ← 5% traffic (canary)

Monitor: error rate, latency for v2
If healthy: 25% → 50% → 100%
If issues: roll back to 0%

Best practice: test on 1% of traffic first
```

### Database Migration with Deployments

```
Zero-downtime migration challenge:
  Old code: uses column 'name'
  New code: uses columns 'first_name' + 'last_name'

Wrong approach:
  1. Rename column → BREAKING CHANGE: old pods crash!

Expand-Contract (safe approach):
  Step 1 (EXPAND): Add new columns, keep old column (both old+new code work)
  Step 2 (MIGRATE): Backfill data from old to new columns
  Step 3 (DEPLOY): Deploy new code (reads new columns)
  Step 4 (CONTRACT): Remove old column once all old pods gone
```

---

## 6. Interview Questions

#### Docker

1. What is the difference between `COPY` and `ADD` in a Dockerfile?
2. Why use multi-stage builds?
3. What does `-XX:+UseContainerSupport` do?

#### Kubernetes

4. What is the difference between `livenessProbe` and `readinessProbe`?
5. What happens when you don't set resource `limits` in Kubernetes?
6. Explain HPA — how does it decide when to scale?

#### CI/CD

7. What is a deployment pipeline? Describe the stages.
8. How do you implement zero-downtime deployments?
9. What is the expand-contract pattern for database migrations?

#### Monitoring

10. What are the three pillars of observability?
11. What metrics would you monitor for a Java microservice?
12. How do you correlate logs across microservices?

---

### Summary — DevOps Cheatsheet

```
Docker:
  Multi-stage build: build tools separate from runtime
  Layer caching: COPY pom.xml first, then src
  Non-root user: RUN adduser, USER appuser
  JVM flags: UseContainerSupport + MaxRAMPercentage=75

Kubernetes:
  Deployment: replicas, rolling update strategy
  Service: ClusterIP (internal), LoadBalancer (cloud)
  ConfigMap: non-sensitive config
  Secret: sensitive data (base64 encoded)
  HPA: auto-scale based on CPU/memory
  Resources: ALWAYS set requests and limits
  Health Probes:
    liveness: restart if dead
    readiness: remove from LB if not ready
    startup: give slow apps time to boot

Deployment Strategies:
  Rolling: gradual, zero-downtime, mixed versions
  Blue-Green: instant switch, instant rollback
  Canary: gradual percentage rollout

Observability (Three Pillars):
  Metrics: Prometheus scrapes /actuator/prometheus → Grafana
  Logs: JSON format + MDC correlation ID → ELK
  Traces: Micrometer Tracing → Zipkin/Jaeger

Key Alerts to Set Up:
  Error rate > 5% for 2 min → CRITICAL
  P99 latency > 2s for 5 min → WARNING
  JVM heap > 85% for 10 min → WARNING
  Kafka consumer lag > 10000 → WARNING
```
