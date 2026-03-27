# Performance Review Guide

Performance review guide covering backend, database, algorithm complexity, and API performance.

## Table of Contents

- [Database Performance](#database-performance)
- [API Performance](#api-performance)
- [Algorithm Complexity](#algorithm-complexity)
- [Memory Management](#memory-management)
- [Concurrency Performance](#concurrency-performance)
- [Performance Review Checklist](#performance-review-checklist)

---

## Database Performance

### N+1 Query Problem

```java
// Bad: N+1 problem - 1 + N queries
List<User> users = userRepository.findAll();  // 1 query
for (User user : users) {
    System.out.println(user.getOrders().size());  // N queries (one per user)
}

// Good: Eager Loading - 2 queries
@Query("SELECT u FROM User u JOIN FETCH u.orders")
List<User> findAllWithOrders();

// Good: EntityGraph
@EntityGraph(attributePaths = {"orders"})
List<User> findAll();

// Good: For Many-to-Many use @BatchSize
@BatchSize(size = 20)
@OneToMany(mappedBy = "user")
private List<Order> orders;
```

### Index Optimization

```sql
-- Bad: Full table scan
SELECT * FROM orders WHERE status = 'pending';

-- Good: Add index
CREATE INDEX idx_orders_status ON orders(status);

-- Bad: Index invalidation due to function operation
SELECT * FROM users WHERE YEAR(created_at) = 2024;

-- Good: Range query can use index
SELECT * FROM users
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01';

-- Bad: Index invalidation due to LIKE prefix wildcard
SELECT * FROM products WHERE name LIKE '%phone%';

-- Good: Prefix match can use index
SELECT * FROM products WHERE name LIKE 'phone%';
```

### Query Optimization

```java
// Bad: SELECT * fetches unnecessary columns
@Query("SELECT * FROM users WHERE id = :id")
User findById(Long id);

// Good: Only query needed columns
@Query("SELECT new com.example.dto.UserSummaryDto(u.id, u.name, u.email) " +
       "FROM User u WHERE u.id = :id")
UserSummaryDto findSummaryById(Long id);

// Bad: Large table without pagination
List<Log> findByType(String type);

// Good: Paginated query
Page<Log> findByType(String type, Pageable pageable);

// Bad: Queries inside loops
for (Long id : userIds) {
    userRepository.findById(id);  // N queries
}

// Good: Batch query
List<User> users = userRepository.findAllById(userIds);  // 1 query
```

### Connection Pool Configuration

```yaml
# Good: Properly configured connection pool
spring:
  datasource:
    hikari:
      minimum-idle: 5
      maximum-pool-size: 20
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
```

### Database Review Checklist

```markdown
Must Check:
- [ ] Are there N+1 queries?
- [ ] Do WHERE clause columns have indexes?
- [ ] Avoiding SELECT *?
- [ ] Do large table queries have pagination?

Should Check:
- [ ] Using EXPLAIN to analyze query plans?
- [ ] Is composite index column order correct?
- [ ] Are there unused indexes?
- [ ] Is there slow query monitoring?
```

---

## API Performance

### Pagination Implementation

```java
// Bad: Returning all data
@GetMapping("/users")
public List<User> getUsers() {
    return userRepository.findAll();  // Could return 100000 records
}

// Good: Pagination + limit max count
@GetMapping("/users")
public Page<UserDto> getUsers(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size) {
    int safeSize = Math.min(size, 100);  // Max 100
    return userService.findAll(PageRequest.of(page, safeSize));
}
```

### Caching Strategy

```java
// Good: Spring Cache example
@Service
public class UserService {
    
    @Cacheable(value = "users", key = "#id")
    public User getUser(Long id) {
        return userRepository.findById(id).orElseThrow();
    }
    
    @CacheEvict(value = "users", key = "#user.id")
    public User updateUser(User user) {
        return userRepository.save(user);
    }
}

// Good: Redis cache with TTL
@Cacheable(value = "users", key = "#id", 
           cacheManager = "redisCacheManager")
public User getUser(Long id) {
    return userRepository.findById(id).orElseThrow();
}

// Cache configuration
@Bean
public RedisCacheConfiguration cacheConfiguration() {
    return RedisCacheConfiguration.defaultCacheConfig()
        .entryTtl(Duration.ofHours(1))
        .disableCachingNullValues();
}
```

### Response Compression

```yaml
# Good: Enable Gzip compression
server:
  compression:
    enabled: true
    mime-types: application/json,application/xml,text/html,text/plain
    min-response-size: 1024
```

```java
// Good: Only return necessary fields
@GetMapping("/users")
public List<UserSummaryDto> getUsers() {
    return userService.findAllSummaries();  // Only id, name, email
}

// Or use JSON Views
public class User {
    @JsonView(Views.Summary.class)
    private Long id;
    
    @JsonView(Views.Summary.class)
    private String name;
    
    @JsonView(Views.Detail.class)
    private String bio;  // Not included in summary
}
```

### Rate Limiting

```java
// Good: Rate limiting with Bucket4j
@Service
public class RateLimiter {
    private final Map<String, Bucket> buckets = new ConcurrentHashMap<>();
    
    public boolean tryConsume(String key) {
        Bucket bucket = buckets.computeIfAbsent(key, k -> 
            Bucket.builder()
                .addLimit(Bandwidth.classic(100, Refill.greedy(100, Duration.ofMinutes(1))))
                .build()
        );
        return bucket.tryConsume(1);
    }
}

@RestController
public class ApiController {
    @Autowired
    private RateLimiter rateLimiter;
    
    @GetMapping("/api/resource")
    public ResponseEntity<?> getResource(HttpServletRequest request) {
        String clientIp = request.getRemoteAddr();
        if (!rateLimiter.tryConsume(clientIp)) {
            return ResponseEntity.status(429)
                .body(Map.of("error", "Too many requests"));
        }
        // Process request
    }
}
```

### API Review Checklist

```markdown
- [ ] Do list endpoints have pagination?
- [ ] Is there a limit on max items per page?
- [ ] Is hot data cached?
- [ ] Is response compression enabled?
- [ ] Is there rate limiting?
- [ ] Only returning necessary fields?
```

---

## Algorithm Complexity

### Common Complexity Comparison

| Complexity | Name | 10 items | 1000 items | 1M items | Example |
|------------|------|----------|------------|----------|---------|
| O(1) | Constant | 1 | 1 | 1 | HashMap lookup |
| O(log n) | Logarithmic | 3 | 10 | 20 | Binary search |
| O(n) | Linear | 10 | 1000 | 1M | Array traversal |
| O(n log n) | Linearithmic | 33 | 10000 | 20M | Quick sort |
| O(n^2) | Quadratic | 100 | 1M | 1T | Nested loops |
| O(2^n) | Exponential | 1024 | infinity | infinity | Recursive fibonacci |

### Code Review Identification

```java
// Bad: O(n^2) - Nested loops
public List<Integer> findDuplicates(int[] arr) {
    List<Integer> duplicates = new ArrayList<>();
    for (int i = 0; i < arr.length; i++) {
        for (int j = i + 1; j < arr.length; j++) {
            if (arr[i] == arr[j]) {
                duplicates.add(arr[i]);
            }
        }
    }
    return duplicates;
}

// Good: O(n) - Use Set
public List<Integer> findDuplicates(int[] arr) {
    Set<Integer> seen = new HashSet<>();
    Set<Integer> duplicates = new LinkedHashSet<>();
    for (int item : arr) {
        if (!seen.add(item)) {
            duplicates.add(item);
        }
    }
    return new ArrayList<>(duplicates);
}
```

```java
// Bad: O(n^2) - contains() called in every loop iteration
public List<Integer> removeDuplicates(List<Integer> list) {
    List<Integer> result = new ArrayList<>();
    for (Integer item : list) {
        if (!result.contains(item)) {  // contains() is O(n)
            result.add(item);
        }
    }
    return result;
}

// Good: O(n) - Use Set
public List<Integer> removeDuplicates(List<Integer> list) {
    return new ArrayList<>(new LinkedHashSet<>(list));
}
```

```java
// Bad: O(n) lookup - traverses every time
List<User> users = loadUsers();

public User getUser(Long id) {
    return users.stream()
        .filter(u -> u.getId().equals(id))
        .findFirst()
        .orElse(null);  // O(n)
}

// Good: O(1) lookup - Use Map
Map<Long, User> userMap = users.stream()
    .collect(Collectors.toMap(User::getId, Function.identity()));

public User getUser(Long id) {
    return userMap.get(id);  // O(1)
}
```

### Space Complexity Considerations

```java
// Warning: O(n) space - Creates new array
int[] doubled = Arrays.stream(arr).map(x -> x * 2).toArray();

// Good: O(1) space - In-place modification (if allowed)
for (int i = 0; i < arr.length; i++) {
    arr[i] *= 2;
}

// Warning: Recursion depth too large may cause StackOverflow
public long factorial(int n) {
    if (n <= 1) return 1;
    return n * factorial(n - 1);  // O(n) stack space
}

// Good: Iterative version O(1) space
public long factorial(int n) {
    long result = 1;
    for (int i = 2; i <= n; i++) {
        result *= i;
    }
    return result;
}
```

### Complexity Review Comments

```markdown
[suggestion] "This nested loop has O(n^2) complexity, may have performance issues with large data"
[blocking] "Array.contains() used in loop here, overall is O(n^2), suggest using Set"
[important] "This recursion depth may cause StackOverflow, suggest converting to iteration"
```

---

## Memory Management

### Common Memory Issues in Java

#### Object Pool Reuse

```java
// Bad: Creating objects frequently in loops
for (int i = 0; i < 1000000; i++) {
    StringBuilder sb = new StringBuilder();
    sb.append("value").append(i);
    process(sb.toString());
}

// Good: Reuse StringBuilder
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 1000000; i++) {
    sb.setLength(0);  // Reset
    sb.append("value").append(i);
    process(sb.toString());
}
```

#### Large Collection Memory

```java
// Bad: Loading all data into memory
List<Order> orders = orderRepository.findAll();  // Millions of records
for (Order order : orders) {
    process(order);
}

// Good: Use Stream processing or pagination
@Query("SELECT o FROM Order o")
Stream<Order> findAllAsStream();

try (Stream<Order> stream = orderRepository.findAllAsStream()) {
    stream.forEach(this::process);
}

// Or batch processing
int pageSize = 1000;
int page = 0;
Page<Order> orderPage;
do {
    orderPage = orderRepository.findAll(PageRequest.of(page++, pageSize));
    orderPage.forEach(this::process);
} while (orderPage.hasNext());
```

#### Connection Leaks

```java
// Bad: Connection not closed
public void process() {
    Connection conn = dataSource.getConnection();
    // If exception here, connection leaks
    Statement stmt = conn.createStatement();
    // ...
}

// Good: try-with-resources
public void process() {
    try (Connection conn = dataSource.getConnection();
         Statement stmt = conn.createStatement()) {
        // ...
    }
}
```

#### ThreadLocal Cleanup

```java
// Bad: ThreadLocal not cleaned up (memory leak in thread pools)
private static final ThreadLocal<User> currentUser = new ThreadLocal<>();

public void setUser(User user) {
    currentUser.set(user);
}

// Good: Always cleanup in finally
public void processRequest(User user) {
    try {
        currentUser.set(user);
        doProcess();
    } finally {
        currentUser.remove();  // Important!
    }
}
```

### Memory Review Checklist

```markdown
- [ ] Are large collections loaded into memory at once?
- [ ] Are database connections properly closed?
- [ ] Are ThreadLocals cleaned up?
- [ ] Are streams properly closed?
- [ ] Are there object creation hotspots in loops?
- [ ] Is there data accumulation in static fields?
```

---

## Concurrency Performance

### Lock Granularity

```java
// Bad: Coarse-grained locking (lock entire method)
public synchronized void process(String key, String value) {
    // Lock held even during I/O operations
    String data = fetchFromRemote(key);  // Network I/O
    cache.put(key, data);
}

// Good: Fine-grained locking
public void process(String key, String value) {
    String data = fetchFromRemote(key);  // No lock during I/O
    synchronized (cache) {  // Only lock when needed
        cache.put(key, data);
    }
}

// Better: Use ConcurrentHashMap
private final ConcurrentHashMap<String, String> cache = new ConcurrentHashMap<>();

public void process(String key, String value) {
    cache.computeIfAbsent(key, k -> fetchFromRemote(k));
}
```

### Async Processing

```java
// Bad: Synchronous processing blocking caller
@PostMapping("/orders")
public Order createOrder(@RequestBody OrderRequest request) {
    Order order = orderService.create(request);
    notificationService.sendEmail(order);  // Slow, blocks response
    analyticsService.track(order);         // Slow, blocks response
    return order;
}

// Good: Async processing
@PostMapping("/orders")
public Order createOrder(@RequestBody OrderRequest request) {
    Order order = orderService.create(request);
    
    // Fire and forget
    CompletableFuture.runAsync(() -> notificationService.sendEmail(order));
    CompletableFuture.runAsync(() -> analyticsService.track(order));
    
    return order;
}

// Better: Use Spring Events
@PostMapping("/orders")
public Order createOrder(@RequestBody OrderRequest request) {
    Order order = orderService.create(request);
    eventPublisher.publishEvent(new OrderCreatedEvent(order));
    return order;
}

@Async
@EventListener
public void handleOrderCreated(OrderCreatedEvent event) {
    notificationService.sendEmail(event.getOrder());
}
```

### Virtual Threads (Java 21+)

```java
// Bad: Traditional thread pool for I/O tasks
ExecutorService executor = Executors.newFixedThreadPool(100);

// Good: Virtual threads for I/O-bound tasks
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();

// Spring Boot 3.2+
// application.yml
spring:
  threads:
    virtual:
      enabled: true
```

---

## Performance Review Checklist

### Must Check (Blocking)

**Database:**
- [ ] Are there N+1 queries?
- [ ] Do list endpoints have pagination?
- [ ] Are there SELECT * on large tables?

**Algorithm:**
- [ ] Are there O(n^2) or worse nested loops?
- [ ] Is contains()/indexOf() called in loops?

**Memory:**
- [ ] Are large datasets loaded entirely into memory?
- [ ] Are resources properly closed (connections, streams)?

### Should Check (Important)

**API:**
- [ ] Is hot data cached?
- [ ] Is response compression enabled?
- [ ] Is rate limiting in place?

**Database:**
- [ ] Are query plans analyzed with EXPLAIN?
- [ ] Are WHERE columns indexed?
- [ ] Is there slow query monitoring?

**Concurrency:**
- [ ] Is lock granularity reasonable?
- [ ] Are I/O operations avoided inside locks?
- [ ] Are virtual threads considered for I/O tasks?

### Nice to Have (Suggestions)

- [ ] Are there JVM performance monitoring tools?
- [ ] Are there performance baseline tests?
- [ ] Is profiling data available?
- [ ] Is CDN used for static resources?

---

## Performance Metrics Thresholds

### Backend Metrics

| Metric | Good | Needs Improvement | Poor |
|--------|------|-------------------|------|
| API Response Time | < 100ms | 100-500ms | > 500ms |
| Database Query | < 50ms | 50-200ms | > 200ms |
| P99 Latency | < 500ms | 500ms-1s | > 1s |
| Thread Pool Utilization | < 70% | 70-90% | > 90% |

### JVM Metrics

| Metric | Good | Warning | Danger |
|--------|------|---------|--------|
| GC Pause Time | < 100ms | 100-500ms | > 500ms |
| Heap Usage | < 70% | 70-85% | > 85% |
| GC Frequency | < 1/min | 1-10/min | > 10/min |

---

## Tool Recommendations

### Profiling Tools

| Tool | Purpose |
|------|---------|
| [VisualVM](https://visualvm.github.io/) | JVM monitoring, heap analysis |
| [JProfiler](https://www.ej-technologies.com/products/jprofiler/overview.html) | CPU/Memory profiling |
| [async-profiler](https://github.com/async-profiler/async-profiler) | Low-overhead sampling profiler |
| [Arthas](https://arthas.aliyun.com/) | Online diagnosis tool |

### Database Analysis

| Tool | Purpose |
|------|---------|
| EXPLAIN | Query plan analysis |
| [p6spy](https://github.com/p6spy/p6spy) | SQL logging with timing |
| [Hibernate Statistics](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#statistics) | JPA query statistics |

### APM Tools

| Tool | Purpose |
|------|---------|
| [Micrometer](https://micrometer.io/) | Metrics collection |
| [Spring Boot Actuator](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html) | Production monitoring |
| [Prometheus + Grafana](https://prometheus.io/) | Metrics visualization |
| [Jaeger](https://www.jaegertracing.io/) / [Zipkin](https://zipkin.io/) | Distributed tracing |
