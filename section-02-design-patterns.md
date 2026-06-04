---
layout: default
title: Design Patterns
nav_order: 3
---

# Section 2: Design Patterns — Complete Guide

> Gang of Four (GoF) patterns + modern Java production usage

---

## Table of Contents

- [Creational Patterns](#creational-patterns)
  - [Singleton](#1-singleton)
  - [Factory Method](#2-factory-method)
  - [Abstract Factory](#3-abstract-factory)
  - [Builder](#4-builder)
  - [Prototype](#5-prototype)
- [Structural Patterns](#structural-patterns)
  - [Adapter](#6-adapter)
  - [Decorator](#7-decorator)
  - [Facade](#8-facade)
  - [Proxy](#9-proxy)
  - [Composite](#10-composite)
- [Behavioral Patterns](#behavioral-patterns)
  - [Strategy](#11-strategy)
  - [Observer](#12-observer)
  - [Command](#13-command)
  - [Template Method](#14-template-method)
  - [Chain of Responsibility](#15-chain-of-responsibility)
  - [State](#16-state)
  - [Mediator](#17-mediator)
- [Pattern Comparisons](#pattern-comparisons)

---

# Creational Patterns

## 1. Singleton

### Definition
Ensures a class has **only one instance** and provides a global access point to it.

### Problem
Database connection pools, configuration managers, logging systems — you want exactly one instance shared across the application.

### Mental Model
The President of a country — there's exactly one at a time, and everyone refers to that one person.

### Implementation (Thread-Safe)

```java
// BEST APPROACH: Enum Singleton (Joshua Bloch recommended)
public enum DatabaseConnectionPool {
    INSTANCE;
    
    private final HikariDataSource dataSource;
    
    DatabaseConnectionPool() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:postgresql://localhost:5432/mydb");
        config.setMaximumPoolSize(20);
        this.dataSource = new HikariDataSource(config);
    }
    
    public Connection getConnection() throws SQLException {
        return dataSource.getConnection();
    }
}

// Usage
Connection conn = DatabaseConnectionPool.INSTANCE.getConnection();
```

```java
// SECOND BEST: Double-Checked Locking with volatile
public class ConfigManager {
    private static volatile ConfigManager instance; // volatile prevents instruction reordering
    private final Properties props;
    
    private ConfigManager() {
        props = loadFromFile();
    }
    
    public static ConfigManager getInstance() {
        if (instance == null) {                     // First check (no lock)
            synchronized (ConfigManager.class) {
                if (instance == null) {             // Second check (with lock)
                    instance = new ConfigManager();
                }
            }
        }
        return instance;
    }
}
```

```java
// THIRD: Initialization-on-demand holder (lazy, thread-safe, no sync overhead)
public class AppConfig {
    private AppConfig() { }
    
    private static class Holder {
        static final AppConfig INSTANCE = new AppConfig();
    }
    
    public static AppConfig getInstance() {
        return Holder.INSTANCE; // JVM guarantees class initialization is thread-safe
    }
}
```

### Why Enum Singleton?
1. Thread-safe by JVM guarantee
2. Serialization-safe (prevents new instance on deserialization)
3. Reflection-safe (cannot call private constructor via reflection on enum)

### Pros
- Controlled access to single instance
- Reduced memory footprint for expensive objects

### Cons
- Global state — hard to test (mock)
- Violates Single Responsibility Principle
- Creates hidden dependencies
- Problematic in clustered environments (each node has its own "singleton")

### Production Use Cases
- Spring Beans are singletons by default
- Connection pools (`HikariCP`)
- Thread pools (`Executors`)
- Logger instances (`LoggerFactory.getLogger(...)`)

### Interview Questions
1. How do you make Singleton thread-safe?
2. Why is enum the best Singleton implementation?
3. How do you break a Singleton with reflection? How to prevent it?
4. Is Spring's `@Bean` singleton the same as GoF Singleton? (No — Spring's is per `ApplicationContext`)
5. How would you write a unit test for a class that uses a Singleton?

---

## 2. Factory Method

### Definition
Defines an interface for creating an object, but lets **subclasses decide** which class to instantiate.

### Problem
You need to create objects but don't want to couple the client code to specific implementations.

### Mental Model
A **pizza franchise** — each city franchise (subclass) decides which local ingredients to use, but the franchise manual (interface) defines the pizza-making process.

```java
// Product interface
interface Notification {
    void send(String message, String recipient);
}

// Concrete products
class EmailNotification implements Notification {
    public void send(String message, String recipient) {
        System.out.println("Email to " + recipient + ": " + message);
    }
}

class SMSNotification implements Notification {
    public void send(String message, String recipient) {
        System.out.println("SMS to " + recipient + ": " + message);
    }
}

class PushNotification implements Notification {
    public void send(String message, String recipient) {
        System.out.println("Push to " + recipient + ": " + message);
    }
}

// Factory
class NotificationFactory {
    public static Notification create(String type) {
        return switch (type.toUpperCase()) {
            case "EMAIL" -> new EmailNotification();
            case "SMS"   -> new SMSNotification();
            case "PUSH"  -> new PushNotification();
            default -> throw new IllegalArgumentException("Unknown type: " + type);
        };
    }
}

// Client
Notification n = NotificationFactory.create("EMAIL");
n.send("Order shipped!", "user@example.com");
```

### Pros
- Client code decoupled from concrete types
- Easy to add new types without changing client

### Cons
- May require subclassing / many classes
- More complex for simple cases

### Production Use Cases
- `DriverManager.getConnection()` in JDBC
- `LoggerFactory.getLogger()` in SLF4J
- `Calendar.getInstance()`
- Spring's `BeanFactory`

---

## 3. Abstract Factory

### Definition
Creates **families of related objects** without specifying their concrete classes.

### Mental Model
A **furniture store theme**: Victorian style gives you Victorian chair + Victorian sofa + Victorian table. Modern style gives you Modern chair + Modern sofa + Modern table. You can't mix Victorian chair with Modern table.

```java
// Abstract products
interface Button { void render(); }
interface Checkbox { void render(); }

// Concrete products — Light theme
class LightButton implements Button { public void render() { System.out.println("[ Light Button ]"); } }
class LightCheckbox implements Checkbox { public void render() { System.out.println("☑ Light Checkbox"); } }

// Concrete products — Dark theme
class DarkButton implements Button { public void render() { System.out.println("[ DARK BUTTON ]"); } }
class DarkCheckbox implements Checkbox { public void render() { System.out.println("☑ DARK Checkbox"); } }

// Abstract factory
interface UIFactory {
    Button createButton();
    Checkbox createCheckbox();
}

// Concrete factories
class LightThemeFactory implements UIFactory {
    public Button createButton() { return new LightButton(); }
    public Checkbox createCheckbox() { return new LightCheckbox(); }
}

class DarkThemeFactory implements UIFactory {
    public Button createButton() { return new DarkButton(); }
    public Checkbox createCheckbox() { return new DarkCheckbox(); }
}
```

### Factory vs Abstract Factory

| | Factory Method | Abstract Factory |
|--|----------------|------------------|
| Creates | One product | Family of related products |
| Extensibility | Add product types | Add product families |
| Use When | One varying product | Multiple related products vary together |

---

## 4. Builder

### Definition
Constructs complex objects **step by step**, separating construction from representation.

### Problem
Constructor with 10+ parameters — which argument is which? What if some are optional?

```java
// TELESCOPING CONSTRUCTOR ANTI-PATTERN
new Order("user123", "ELECTRONICS", 1599.99, "USD", true, false, null, null, "EXPRESS", "2026-01-01");
// What is the 6th argument?? Nobody knows.
```

```java
// BUILDER PATTERN
@Builder  // Lombok generates the builder
public class Order {
    private final String userId;
    private final String category;
    private final BigDecimal amount;
    private final String currency;
    private final boolean requiresShipping;
    private final String shippingAddress;
    private final String deliveryType;
    private final LocalDate deliveryDate;
}

// Clean and readable
Order order = Order.builder()
    .userId("user123")
    .category("ELECTRONICS")
    .amount(new BigDecimal("1599.99"))
    .currency("USD")
    .requiresShipping(true)
    .shippingAddress("123 Main St, NY")
    .deliveryType("EXPRESS")
    .deliveryDate(LocalDate.of(2026, 1, 1))
    .build();
```

### Manual Builder (for understanding)

```java
public class HttpRequest {
    private final String url;
    private final String method;
    private final Map<String, String> headers;
    private final String body;
    private final int timeoutMs;
    
    private HttpRequest(Builder builder) {
        this.url = builder.url;
        this.method = builder.method;
        this.headers = Collections.unmodifiableMap(builder.headers);
        this.body = builder.body;
        this.timeoutMs = builder.timeoutMs;
    }
    
    public static class Builder {
        private final String url;
        private String method = "GET";
        private Map<String, String> headers = new HashMap<>();
        private String body;
        private int timeoutMs = 30000;
        
        public Builder(String url) { this.url = url; } // required parameter
        
        public Builder method(String method) { this.method = method; return this; }
        public Builder header(String key, String value) { headers.put(key, value); return this; }
        public Builder body(String body) { this.body = body; return this; }
        public Builder timeout(int ms) { this.timeoutMs = ms; return this; }
        
        public HttpRequest build() {
            if (url == null || url.isBlank()) throw new IllegalStateException("URL required");
            return new HttpRequest(this);
        }
    }
}

// Usage
HttpRequest req = new HttpRequest.Builder("https://api.example.com/orders")
    .method("POST")
    .header("Authorization", "Bearer token123")
    .header("Content-Type", "application/json")
    .body("{\"amount\": 100}")
    .timeout(5000)
    .build();
```

### Builder vs Factory

| | Builder | Factory |
|--|---------|---------|
| Focus | Complex construction, many optional params | Which type to create |
| Returns | Same type always | Different subtypes |
| Readability | High (named params) | Medium |
| Use When | Complex object with many optional fields | Multiple subtypes, decoupled creation |

### Production Use Cases
- Lombok `@Builder` everywhere in Spring apps
- `RestTemplate`, `WebClient` configuration
- `MockMvc` test setup
- SQL query builders (JOOQ, QueryDSL)
- `StringBuilder`

---

## 5. Prototype

### Definition
Creates new objects by **cloning** an existing object.

### Use When
Object creation is expensive (DB query, API call, complex computation) and you want copies.

```java
public class UserProfile implements Cloneable {
    private String userId;
    private List<String> permissions;
    private Map<String, String> metadata;
    
    @Override
    public UserProfile clone() {
        try {
            UserProfile clone = (UserProfile) super.clone();
            clone.permissions = new ArrayList<>(this.permissions); // deep copy list
            clone.metadata = new HashMap<>(this.metadata);         // deep copy map
            return clone;
        } catch (CloneNotSupportedException e) {
            throw new RuntimeException(e);
        }
    }
}

// Prototype registry
class ProfileRegistry {
    private Map<String, UserProfile> registry = new HashMap<>();
    
    public void register(String role, UserProfile profile) {
        registry.put(role, profile);
    }
    
    public UserProfile get(String role) {
        return registry.get(role).clone(); // return clone, not original
    }
}
```

> **Production Note:** Prefer copy constructors or serialization-based cloning over `Cloneable`. The `Cloneable` interface is considered broken by Joshua Bloch.

---

# Structural Patterns

## 6. Adapter

### Definition
Converts the interface of a class into another interface that clients expect. Makes incompatible interfaces work together.

### Mental Model
A **power adapter** for your laptop in a foreign country — the plug shape differs but the functionality (electricity) is the same.

```java
// Existing (incompatible) third-party payment library
class LegacyPaymentGateway {
    public boolean chargeCard(String cardNumber, double amount, String currency) {
        // old API
        return true;
    }
}

// Interface our application expects
interface PaymentProcessor {
    PaymentResult processPayment(PaymentRequest request);
}

// Adapter bridges the gap
class LegacyPaymentAdapter implements PaymentProcessor {
    private final LegacyPaymentGateway legacy;
    
    public LegacyPaymentAdapter(LegacyPaymentGateway legacy) {
        this.legacy = legacy;
    }
    
    @Override
    public PaymentResult processPayment(PaymentRequest request) {
        boolean success = legacy.chargeCard(
            request.getCardNumber(),
            request.getAmount().doubleValue(),
            request.getCurrency()
        );
        return success ? PaymentResult.success() : PaymentResult.failure("Charge failed");
    }
}
```

### Production Use Cases
- Spring's `HandlerAdapter` (adapts various controller types to handler interface)
- JDBC adapters for different database drivers
- Wrapping AWS SDK, Stripe SDK in internal interfaces

---

## 7. Decorator

### Definition
Attaches **additional responsibilities** to an object dynamically, as an alternative to subclassing.

### Mental Model
**Coffee customization**: Start with espresso, wrap with milk (MilkDecorator), wrap with vanilla (VanillaDecorator). Each wrapper adds behavior without modifying the original.

```java
// Component interface
interface DataSource {
    void writeData(String data);
    String readData();
}

// Concrete component
class FileDataSource implements DataSource {
    private final String filename;
    
    public FileDataSource(String filename) { this.filename = filename; }
    
    public void writeData(String data) { /* write to file */ }
    public String readData() { return /* read from file */ "data"; }
}

// Base decorator
abstract class DataSourceDecorator implements DataSource {
    protected final DataSource wrapped;
    
    DataSourceDecorator(DataSource source) { this.wrapped = source; }
    
    public void writeData(String data) { wrapped.writeData(data); }
    public String readData() { return wrapped.readData(); }
}

// Concrete decorators
class EncryptionDecorator extends DataSourceDecorator {
    EncryptionDecorator(DataSource source) { super(source); }
    
    public void writeData(String data) {
        super.writeData(encrypt(data)); // encrypt before writing
    }
    public String readData() {
        return decrypt(super.readData()); // decrypt after reading
    }
}

class CompressionDecorator extends DataSourceDecorator {
    CompressionDecorator(DataSource source) { super(source); }
    
    public void writeData(String data) {
        super.writeData(compress(data));
    }
    public String readData() {
        return decompress(super.readData());
    }
}

// Usage — stack decorators
DataSource source = new CompressionDecorator(
                        new EncryptionDecorator(
                            new FileDataSource("data.bin")
                        )
                    );
source.writeData("sensitive data"); // compress(encrypt(write))
```

### Decorator vs Proxy vs Inheritance

| | Decorator | Proxy | Inheritance |
|--|-----------|-------|-------------|
| Purpose | Add behavior dynamically | Control access | Extend type |
| Wraps Same Interface | Yes | Yes | No |
| Multiple additions | Stack decorators | Usually one proxy | Deep hierarchy |
| Runtime flexibility | Yes | Limited | No (compile-time) |
| Spring Usage | `@Transactional`, `@Cacheable` | CGLIB/JDK proxy for AOP | Java class hierarchy |

---

## 8. Facade

### Definition
Provides a **simplified interface** to a complex subsystem.

### Mental Model
A **hotel concierge** — you ask them "arrange a dinner for me", and they coordinate reservations, transport, and booking. You don't deal with each subsystem.

```java
// Complex subsystems
class OrderValidationService {
    public void validate(Order order) { /* complex validation */ }
}

class InventoryService {
    public void reserve(List<OrderItem> items) { /* check and reserve stock */ }
}

class PaymentService {
    public PaymentResult charge(Payment payment) { /* charge customer */ return null; }
}

class ShippingService {
    public Shipment createShipment(Order order) { /* create shipment */ return null; }
}

class NotificationService {
    public void notifyCustomer(String userId, String message) { /* send email/push */ }
}

// FACADE — simplified interface for checkout
@Service
public class CheckoutFacade {
    private final OrderValidationService validator;
    private final InventoryService inventory;
    private final PaymentService payment;
    private final ShippingService shipping;
    private final NotificationService notification;
    
    public CheckoutResult checkout(CheckoutRequest request) {
        validator.validate(request.getOrder());
        inventory.reserve(request.getOrder().getItems());
        PaymentResult paymentResult = payment.charge(request.getPayment());
        Shipment shipment = shipping.createShipment(request.getOrder());
        notification.notifyCustomer(request.getUserId(), "Order confirmed: " + shipment.getTrackingId());
        
        return CheckoutResult.success(shipment.getTrackingId());
    }
}
```

### Production Use Cases
- Service layer in Spring MVC (facade over repositories + domain services)
- `SLF4J` facade over Log4j/Logback
- Spring's `JdbcTemplate` (facade over JDBC)

---

## 9. Proxy

### Definition
Provides a **surrogate** object that controls access to the real object.

### Types

| Proxy Type | Purpose | Example |
|-----------|---------|---------|
| **Virtual Proxy** | Lazy initialization | Hibernate lazy loading |
| **Protection Proxy** | Access control | Spring Security proxy |
| **Remote Proxy** | Network transparency | OpenFeign client |
| **Caching Proxy** | Cache results | Spring `@Cacheable` |
| **Logging Proxy** | Log calls | AOP logging aspect |

```java
// Protection proxy example
interface BankAccount {
    void transfer(BigDecimal amount, String toAccount);
    BigDecimal getBalance();
}

class RealBankAccount implements BankAccount {
    private BigDecimal balance;
    
    public void transfer(BigDecimal amount, String toAccount) { balance = balance.subtract(amount); }
    public BigDecimal getBalance() { return balance; }
}

class BankAccountProxy implements BankAccount {
    private final RealBankAccount real;
    private final User currentUser;
    
    public void transfer(BigDecimal amount, String toAccount) {
        if (!currentUser.hasPermission("TRANSFER")) {
            throw new SecurityException("Transfer not permitted");
        }
        if (amount.compareTo(new BigDecimal("10000")) > 0) {
            requireTwoFactorAuth(); // additional check for large transfers
        }
        real.transfer(amount, toAccount);
        auditLog("TRANSFER", currentUser, amount, toAccount);
    }
    
    public BigDecimal getBalance() {
        if (!currentUser.hasPermission("READ_BALANCE")) throw new SecurityException();
        return real.getBalance();
    }
}
```

### How Spring AOP Uses Proxy

```
@Transactional → Spring creates CGLIB proxy wrapping your bean
When you call myService.save(), you're actually calling:
  TransactionProxy.save()
    → opens transaction
    → calls real myService.save()
    → commits or rolls back
    → returns result
```

---

## 10. Composite

### Definition
Composes objects into **tree structures** to represent part-whole hierarchies.

```java
// Component
interface FileSystemItem {
    String getName();
    long getSize();
    void print(String indent);
}

// Leaf
class File implements FileSystemItem {
    private final String name;
    private final long size;
    
    public String getName() { return name; }
    public long getSize() { return size; }
    public void print(String indent) {
        System.out.println(indent + "📄 " + name + " (" + size + " bytes)");
    }
}

// Composite
class Directory implements FileSystemItem {
    private final String name;
    private final List<FileSystemItem> children = new ArrayList<>();
    
    public void add(FileSystemItem item) { children.add(item); }
    public void remove(FileSystemItem item) { children.remove(item); }
    
    public String getName() { return name; }
    public long getSize() {
        return children.stream().mapToLong(FileSystemItem::getSize).sum();
    }
    public void print(String indent) {
        System.out.println(indent + "📁 " + name);
        children.forEach(c -> c.print(indent + "  "));
    }
}
```

---

# Behavioral Patterns

## 11. Strategy

### Definition
Defines a family of algorithms, encapsulates each one, and makes them **interchangeable** at runtime.

### Mental Model
**GPS navigation** — you can switch between Fastest Route, Shortest Route, or Avoid Highways. The destination doesn't change; only the algorithm to get there changes.

```java
// Strategy interface
interface SortStrategy {
    void sort(int[] array);
}

// Concrete strategies
class QuickSort implements SortStrategy {
    public void sort(int[] array) { /* quicksort implementation */ }
}
class MergeSort implements SortStrategy {
    public void sort(int[] array) { /* mergesort implementation */ }
}
class BubbleSort implements SortStrategy { // for small arrays
    public void sort(int[] array) { /* bubblesort */ }
}

// Context
class DataProcessor {
    private SortStrategy strategy;
    
    public DataProcessor(SortStrategy strategy) {
        this.strategy = strategy;
    }
    
    public void setStrategy(SortStrategy strategy) { // swap at runtime
        this.strategy = strategy;
    }
    
    public void process(int[] data) {
        strategy.sort(data);
    }
}

// Production example: payment strategy
interface PaymentStrategy {
    PaymentResult pay(BigDecimal amount, PaymentDetails details);
}

@Component("creditCard") class CreditCardStrategy implements PaymentStrategy { ... }
@Component("upi")        class UPIStrategy implements PaymentStrategy { ... }
@Component("netBanking") class NetBankingStrategy implements PaymentStrategy { ... }

@Service
class CheckoutService {
    @Autowired
    private Map<String, PaymentStrategy> strategies; // Spring injects all implementations
    
    public PaymentResult checkout(Order order, String paymentMethod) {
        PaymentStrategy strategy = strategies.get(paymentMethod);
        if (strategy == null) throw new IllegalArgumentException("Unknown payment method");
        return strategy.pay(order.getTotal(), order.getPaymentDetails());
    }
}
```

### Strategy vs Factory

| | Strategy | Factory |
|--|----------|---------|
| Focus | **How** to do something (algorithm) | **What** to create (object) |
| Runtime swap | Yes | Typically no |
| Contains logic | Yes | No (creates objects) |
| Use When | Algorithm varies | Object type varies |

---

## 12. Observer

### Definition
Defines a one-to-many dependency: when one object changes state, **all dependents are notified automatically**.

### Mental Model
**YouTube subscription** — when a creator posts, all subscribers get notified. Creator doesn't know subscribers; subscribers don't know each other.

```java
// Observer interface
interface OrderEventListener {
    void onOrderEvent(OrderEvent event);
}

// Subject (Observable)
class OrderService {
    private final List<OrderEventListener> listeners = new ArrayList<>();
    
    public void subscribe(OrderEventListener listener) { listeners.add(listener); }
    public void unsubscribe(OrderEventListener listener) { listeners.remove(listener); }
    
    public Order placeOrder(OrderRequest request) {
        Order order = createOrder(request);
        notifyListeners(new OrderEvent(OrderEventType.ORDER_PLACED, order));
        return order;
    }
    
    private void notifyListeners(OrderEvent event) {
        listeners.forEach(l -> l.onOrderEvent(event));
    }
}

// Concrete observers
class InventoryUpdateListener implements OrderEventListener {
    public void onOrderEvent(OrderEvent event) {
        if (event.getType() == ORDER_PLACED) {
            reserveInventory(event.getOrder());
        }
    }
}

class EmailNotificationListener implements OrderEventListener {
    public void onOrderEvent(OrderEvent event) {
        sendConfirmationEmail(event.getOrder().getUserId());
    }
}
```

### Spring's Event System (Observer built-in)

```java
// Event class
public class OrderPlacedEvent extends ApplicationEvent {
    private final Order order;
    public OrderPlacedEvent(Object source, Order order) {
        super(source);
        this.order = order;
    }
}

// Publisher
@Service
public class OrderService {
    @Autowired
    private ApplicationEventPublisher publisher;
    
    public Order placeOrder(OrderRequest request) {
        Order order = createOrder(request);
        publisher.publishEvent(new OrderPlacedEvent(this, order));
        return order;
    }
}

// Listeners
@Component
public class InventoryListener {
    @EventListener
    public void handle(OrderPlacedEvent event) {
        reserveInventory(event.getOrder());
    }
}

@Component
public class EmailListener {
    @EventListener
    @Async  // handle asynchronously
    public void handle(OrderPlacedEvent event) {
        sendEmail(event.getOrder());
    }
}
```

### Observer vs Pub/Sub

| | Observer | Pub/Sub |
|--|----------|---------|
| Coupling | Subject knows observer interface | Publisher doesn't know subscribers |
| Channel | Direct | Via message broker (Kafka, RabbitMQ) |
| Async | Optional | Usually async |
| Distributed | Same JVM | Cross-service |
| Example | Spring Events | Kafka events |

---

## 13. Command

### Definition
Encapsulates a request as an object, allowing you to **queue, log, undo, and retry** operations.

```java
// Command interface
interface Command {
    void execute();
    void undo();
}

// Concrete commands
class TransferMoneyCommand implements Command {
    private final Account from, to;
    private final BigDecimal amount;
    
    public void execute() {
        from.debit(amount);
        to.credit(amount);
    }
    
    public void undo() { // rollback
        to.debit(amount);
        from.credit(amount);
    }
}

// Command processor with history
class CommandProcessor {
    private final Deque<Command> history = new ArrayDeque<>();
    
    public void execute(Command cmd) {
        cmd.execute();
        history.push(cmd);
    }
    
    public void undoLast() {
        if (!history.isEmpty()) {
            history.pop().undo();
        }
    }
}
```

### Production Use Cases
- Database transaction undo logs
- Browser "back" button
- Game move history
- Job queues (each job = command)

---

## 14. Template Method

### Definition
Defines the **skeleton of an algorithm** in a base class, deferring some steps to subclasses.

### Mental Model
A **recipe book** defines the process (prepare → cook → plate → serve), but each recipe fills in different ingredients.

```java
// Abstract template
abstract class DataImporter {
    // Template method — defines the algorithm skeleton
    public final void importData(String source) {
        String raw = readData(source);      // step 1
        String validated = validateData(raw); // step 2
        String transformed = transformData(validated); // step 3
        saveToDatabase(transformed);         // step 4
        sendImportReport();                  // step 5 (optional hook)
    }
    
    protected abstract String readData(String source);
    protected abstract String validateData(String data);
    protected abstract String transformData(String data);
    
    protected void saveToDatabase(String data) { /* default impl */ }
    
    protected void sendImportReport() {
        // hook — subclasses can override or leave as-is
    }
}

class CSVImporter extends DataImporter {
    protected String readData(String filePath) { return readCSVFile(filePath); }
    protected String validateData(String data) { return validateCSVSchema(data); }
    protected String transformData(String data) { return parseCSVToJSON(data); }
}

class XMLImporter extends DataImporter {
    protected String readData(String url) { return fetchXMLFromURL(url); }
    protected String validateData(String data) { return validateXMLAgainstXSD(data); }
    protected String transformData(String data) { return parseXMLToJSON(data); }
}
```

### Spring Usage
- `JdbcTemplate` — template for DB operations
- `RestTemplate` — template for HTTP calls
- `AbstractSecurityInterceptor` — template for security checks

---

## 15. Chain of Responsibility

### Definition
Passes a request along a **chain of handlers**, where each handler decides to process or pass along.

### Mental Model
**Customer support escalation** — L1 → L2 → L3 → Manager. Each level tries to handle; if it can't, escalates.

```java
abstract class RequestHandler {
    private RequestHandler next;
    
    public RequestHandler setNext(RequestHandler next) {
        this.next = next;
        return next;
    }
    
    public abstract boolean handle(HttpRequest request);
    
    protected boolean passToNext(HttpRequest request) {
        if (next != null) return next.handle(request);
        return false;
    }
}

class AuthenticationHandler extends RequestHandler {
    public boolean handle(HttpRequest request) {
        if (!isAuthenticated(request)) {
            sendUnauthorized(request);
            return false;
        }
        return passToNext(request);
    }
}

class AuthorizationHandler extends RequestHandler {
    public boolean handle(HttpRequest request) {
        if (!isAuthorized(request)) {
            sendForbidden(request);
            return false;
        }
        return passToNext(request);
    }
}

class RateLimitHandler extends RequestHandler {
    public boolean handle(HttpRequest request) {
        if (isRateLimited(request)) {
            sendTooManyRequests(request);
            return false;
        }
        return passToNext(request);
    }
}

// Setup chain
RequestHandler chain = new AuthenticationHandler();
chain.setNext(new AuthorizationHandler())
     .setNext(new RateLimitHandler());
// Spring Filter Chain is exactly this pattern!
```

---

## 16. State

### Definition
Allows an object to **alter its behavior when its internal state changes**. The object will appear to change its class.

```java
// Order state machine
interface OrderState {
    void confirm(OrderContext ctx);
    void ship(OrderContext ctx);
    void deliver(OrderContext ctx);
    void cancel(OrderContext ctx);
}

class PendingState implements OrderState {
    public void confirm(OrderContext ctx) { ctx.setState(new ConfirmedState()); }
    public void ship(OrderContext ctx) { throw new IllegalStateException("Must confirm first"); }
    public void deliver(OrderContext ctx) { throw new IllegalStateException("Must ship first"); }
    public void cancel(OrderContext ctx) { ctx.setState(new CancelledState()); }
}

class ConfirmedState implements OrderState {
    public void confirm(OrderContext ctx) { /* already confirmed */ }
    public void ship(OrderContext ctx) { ctx.setState(new ShippedState()); }
    public void deliver(OrderContext ctx) { throw new IllegalStateException(); }
    public void cancel(OrderContext ctx) { ctx.setState(new CancelledState()); }
}

class OrderContext {
    private OrderState state = new PendingState();
    
    public void setState(OrderState state) { this.state = state; }
    public void confirm() { state.confirm(this); }
    public void ship() { state.ship(this); }
    public void deliver() { state.deliver(this); }
    public void cancel() { state.cancel(this); }
}
```

---

## 17. Mediator

### Definition
Defines an object that **encapsulates communication** between multiple objects, reducing direct dependencies.

```java
// Chat room mediator
interface ChatMediator {
    void sendMessage(String message, User sender);
    void addUser(User user);
}

class ChatRoom implements ChatMediator {
    private List<User> users = new ArrayList<>();
    
    public void addUser(User user) { users.add(user); }
    
    public void sendMessage(String message, User sender) {
        users.stream()
            .filter(u -> !u.equals(sender))
            .forEach(u -> u.receive(message, sender.getName()));
    }
}
```

---

# Pattern Comparisons

## Strategy vs Factory

| Dimension | Strategy | Factory |
|-----------|----------|---------|
| **Intent** | Vary the *algorithm* at runtime | Vary the *object type* at creation |
| **Has logic** | Yes (algorithm) | No (just creates) |
| **Runtime swap** | Yes | Usually no |
| **Example** | Payment method selection | Notification channel creation |

## Decorator vs Proxy

| Dimension | Decorator | Proxy |
|-----------|-----------|-------|
| **Intent** | Add/enhance behavior | Control access / surrogate |
| **Client aware** | Usually | Usually not |
| **Stacking** | Common (multiple decorators) | Usually one proxy |
| **Examples** | InputStream wrappers | CGLIB proxies, Hibernate lazy load |

## Builder vs Factory

| Dimension | Builder | Factory |
|-----------|---------|---------|
| **Focus** | How to *construct* (step by step) | What *type* to create |
| **Complexity** | Complex objects, many params | Simple to medium objects |
| **Returns** | Same type | Different subtypes |
| **Examples** | `HttpRequest.Builder`, `StringBuilder` | `Calendar.getInstance()` |

## Observer vs Pub/Sub

| Dimension | Observer | Pub/Sub |
|-----------|----------|---------|
| **Coupling** | Subject knows observer interface | Zero coupling via broker |
| **Scope** | Usually same process | Cross-process, distributed |
| **Ordering** | Synchronous (usually) | Asynchronous |
| **Examples** | Spring `ApplicationEvent` | Kafka, RabbitMQ, SNS |

## Composition vs Inheritance

| | Composition | Inheritance |
|--|-------------|-------------|
| Relationship | HAS-A | IS-A |
| Coupling | Low | High (fragile base class) |
| Flexibility | Swap at runtime | Fixed at compile time |
| Testing | Easy to mock | Hard to isolate |
| Principle | Favor composition | Use only for true IS-A |
| Example | `OrderService` has `PaymentStrategy` | `Square` extends `Rectangle` (LSP violation!) |

---

## Summary — Design Patterns Cheatsheet

```
CREATIONAL
├── Singleton   → One instance, enum is best
├── Factory     → Decouple creation from client, returns subtype
├── Abs Factory → Family of related objects
├── Builder     → Complex objects, readable construction, Lombok @Builder
└── Prototype   → Clone expensive objects

STRUCTURAL
├── Adapter     → Make incompatible interfaces work (old API wrapper)
├── Decorator   → Add behavior without subclassing (InputStream, AOP)
├── Facade      → Simplify complex subsystem (Service layer)
├── Proxy       → Control access, AOP (CGLIB, JDK Proxy)
└── Composite   → Tree structures, uniform treatment (FileSystem)

BEHAVIORAL
├── Strategy    → Swap algorithms (payment methods, sort strategies)
├── Observer    → Notify dependents (Spring Events, Kafka)
├── Command     → Encapsulate request (undo/redo, job queue)
├── Template    → Algorithm skeleton (JdbcTemplate)
├── Chain       → Pass request along handlers (Filter Chain)
├── State       → State machine behavior (Order status)
└── Mediator    → Centralized communication (Chat room)

KEY COMPARISONS
├── Strategy vs Factory: algorithm variation vs type variation
├── Decorator vs Proxy: enhancement vs access control
├── Builder vs Factory: construction vs creation
└── Observer vs PubSub: direct vs broker-mediated
```
