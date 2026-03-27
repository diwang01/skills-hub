# Java Code Review Guide

Java code review focus: Java 17/21 new features, Spring Boot 3 best practices, concurrent programming (virtual threads), JPA performance optimization, and code maintainability.

## Table of Contents

- [Modern Java Features (17/21+)](#modern-java-features-1721)
- [Stream API & Optional](#stream-api--optional)
- [Spring Boot Best Practices](#spring-boot-best-practices)
- [JPA & Database Performance](#jpa--database-performance)
- [Concurrency & Virtual Threads](#concurrency--virtual-threads)
- [Lombok Usage Guidelines](#lombok-usage-guidelines)
- [Exception Handling](#exception-handling)
- [Testing Guidelines](#testing-guidelines)
- [Review Checklist](#review-checklist)

---

## Modern Java Features (17/21+)

### Record (Record Classes)

```java
// Bad: Traditional POJO/DTO with boilerplate code
public class UserDto {
    private final String name;
    private final int age;

    public UserDto(String name, int age) {
        this.name = name;
        this.age = age;
    }
    // getters, equals, hashCode, toString...
}

// Good: Use Record - concise, immutable, semantically clear
public record UserDto(String name, int age) {
    // Compact constructor for validation
    public UserDto {
        if (age < 0) throw new IllegalArgumentException("Age cannot be negative");
    }
}
```

### Switch Expressions & Pattern Matching

```java
// Bad: Traditional Switch - easy to miss break, verbose and error-prone
String type = "";
switch (obj) {
    case Integer i: // Java 16+
        type = String.format("int %d", i);
        break;
    case String s:
        type = String.format("string %s", s);
        break;
    default:
        type = "unknown";
}

// Good: Switch expression - no fall-through risk, enforced return value
String type = switch (obj) {
    case Integer i -> "int %d".formatted(i);
    case String s  -> "string %s".formatted(s);
    case null      -> "null value"; // Java 21 null handling
    default        -> "unknown";
};
```

### Text Blocks

```java
// Bad: String concatenation for SQL/JSON
String json = "{\n" +
              "  \"name\": \"Alice\",\n" +
              "  \"age\": 20\n" +
              "}";

// Good: Use text blocks - WYSIWYG
String json = """
    {
      "name": "Alice",
      "age": 20
    }
    """;
```

---

## Stream API & Optional

### Avoid Stream Abuse

```java
// Bad: Simple loop doesn't need Stream (performance overhead + poor readability)
items.stream().forEach(item -> {
    process(item);
});

// Good: Use for-each for simple scenarios
for (var item : items) {
    process(item);
}

// Bad: Extremely complex Stream chain
List<Dto> result = list.stream()
    .filter(...)
    .map(...)
    .peek(...)
    .sorted(...)
    .collect(...); // Hard to debug

// Good: Split into meaningful steps
var filtered = list.stream().filter(...).toList();
// ...
```

### Correct Optional Usage

```java
// Bad: Using Optional as parameter or field (serialization issues, adds call complexity)
public void process(Optional<String> name) { ... }
public class User {
    private Optional<String> email; // Not recommended
}

// Good: Optional only for return values
public Optional<User> findUser(String id) { ... }

// Bad: Using isPresent() + get() with Optional
Optional<User> userOpt = findUser(id);
if (userOpt.isPresent()) {
    return userOpt.get().getName();
} else {
    return "Unknown";
}

// Good: Use functional API
return findUser(id)
    .map(User::getName)
    .orElse("Unknown");
```

---

## Spring Boot Best Practices

### Dependency Injection (DI)

```java
// Bad: Field injection (@Autowired)
// Cons: Hard to test (requires reflection), hides excessive dependencies, poor immutability
@Service
public class UserService {
    @Autowired
    private UserRepository userRepo;
}

// Good: Constructor Injection
// Pros: Clear dependencies, easy unit testing (Mock), fields can be final
@Service
public class UserService {
    private final UserRepository userRepo;

    public UserService(UserRepository userRepo) {
        this.userRepo = userRepo;
    }
}
// Tip: Combine with Lombok @RequiredArgsConstructor to simplify code, but watch for circular dependencies
```

### Configuration Management

```java
// Bad: Hardcoded configuration values
@Service
public class PaymentService {
    private String apiKey = "sk_live_12345";
}

// Bad: @Value scattered throughout code
@Value("${app.payment.api-key}")
private String apiKey;

// Good: Type-safe configuration with @ConfigurationProperties
@ConfigurationProperties(prefix = "app.payment")
public record PaymentProperties(String apiKey, int timeout, String url) {}
```

---

## JPA & Database Performance

### N+1 Query Problem

```java
// Bad: FetchType.EAGER or triggering lazy load in loop
// Entity definition
@Entity
public class User {
    @OneToMany(fetch = FetchType.EAGER) // Dangerous!
    private List<Order> orders;
}

// Business code
List<User> users = userRepo.findAll(); // 1 SQL query
for (User user : users) {
    // If Lazy, this triggers N SQL queries
    System.out.println(user.getOrders().size());
}

// Good: Use @EntityGraph or JOIN FETCH
@Query("SELECT u FROM User u JOIN FETCH u.orders")
List<User> findAllWithOrders();
```

### Transaction Management

```java
// Bad: Opening transaction at Controller layer (database connection held too long)
// Bad: @Transactional on private methods (AOP doesn't work)
@Transactional
private void saveInternal() { ... }

// Good: @Transactional on public Service layer methods
// Good: Mark read operations with readOnly = true (performance optimization)
@Service
public class UserService {
    @Transactional(readOnly = true)
    public User getUser(Long id) { ... }

    @Transactional
    public void createUser(UserDto dto) { ... }
}
```

### Entity Design

```java
// Bad: Using Lombok @Data on Entity
// @Data generates equals/hashCode including all fields, may trigger lazy loading causing performance issues or exceptions
@Entity
@Data
public class User { ... }

// Good: Use only @Getter, @Setter
// Good: Custom equals/hashCode (usually based on ID)
@Entity
@Getter
@Setter
public class User {
    @Id
    private Long id;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof User)) return false;
        return id != null && id.equals(((User) o).id);
    }

    @Override
    public int hashCode() {
        return getClass().hashCode();
    }
}
```

---

## Concurrency & Virtual Threads

### Virtual Threads (Java 21+)

```java
// Bad: Traditional thread pool for large I/O blocking tasks (resource exhaustion)
ExecutorService executor = Executors.newFixedThreadPool(100);

// Good: Use virtual threads for I/O-intensive tasks (high throughput)
// Spring Boot 3.2+ enable: spring.threads.virtual.enabled=true
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();

// With virtual threads, blocking operations (like DB queries, HTTP requests) consume almost no OS thread resources
```

### Thread Safety

```java
// Bad: SimpleDateFormat is not thread-safe
private static final SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");

// Good: Use DateTimeFormatter (Java 8+)
private static final DateTimeFormatter dtf = DateTimeFormatter.ofPattern("yyyy-MM-dd");

// Bad: HashMap in multi-threaded environment may cause infinite loop or data loss
// Good: Use ConcurrentHashMap
Map<String, String> cache = new ConcurrentHashMap<>();
```

---

## Lombok Usage Guidelines

```java
// Bad: Abusing @Builder makes it impossible to enforce required field validation
@Builder
public class Order {
    private String id; // Required
    private String note; // Optional
}
// Caller may miss id: Order.builder().note("hi").build();

// Good: For critical business objects, manually write Builder or constructor to ensure invariants
// Or add validation in build() method (Lombok @Builder.Default etc.)
```

---

## Exception Handling

### Global Exception Handling

```java
// Bad: try-catch everywhere swallowing exceptions or only printing logs
try {
    userService.create(user);
} catch (Exception e) {
    e.printStackTrace(); // Should not be used in production
    // return null; // Swallows exception, caller doesn't know what happened
}

// Good: Custom exception + @ControllerAdvice (Spring Boot 3 ProblemDetail)
public class UserNotFoundException extends RuntimeException { ... }

@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(UserNotFoundException.class)
    public ProblemDetail handleNotFound(UserNotFoundException e) {
        return ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, e.getMessage());
    }
}
```

---

## Testing Guidelines

### Unit Tests vs Integration Tests

```java
// Bad: Unit tests depending on real database or external services
@SpringBootTest // Starts entire Context, slow
public class UserServiceTest { ... }

// Good: Unit tests use Mockito
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    @Mock UserRepository repo;
    @InjectMocks UserService service;

    @Test
    void shouldCreateUser() { ... }
}

// Good: Integration tests use Testcontainers
@Testcontainers
@SpringBootTest
class UserRepositoryTest {
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");
    // ...
}
```

---

## Review Checklist

### Basics & Standards
- [ ] Follow Java 17/21 new features (Switch expressions, Records, Text blocks)
- [ ] Avoid deprecated classes (Date, Calendar, SimpleDateFormat)
- [ ] Collection operations use Stream API or Collections methods where appropriate?
- [ ] Optional only used for return values, not for fields or parameters

### Spring Boot
- [ ] Use constructor injection instead of @Autowired field injection
- [ ] Configuration properties use @ConfigurationProperties
- [ ] Controller has single responsibility, business logic moved to Service
- [ ] Global exception handling uses @ControllerAdvice / ProblemDetail

### Database & Transactions
- [ ] Read operations marked with `@Transactional(readOnly = true)`
- [ ] Check for N+1 queries (EAGER fetch or loop calls)
- [ ] Entity classes don't use @Data, correctly implement equals/hashCode
- [ ] Database indexes cover query conditions

### Concurrency & Performance
- [ ] I/O-intensive tasks consider virtual threads?
- [ ] Thread-safe classes used correctly (ConcurrentHashMap vs HashMap)
- [ ] Lock granularity reasonable? Avoid I/O operations inside locks

### Maintainability
- [ ] Critical business logic has sufficient unit tests
- [ ] Proper logging (use Slf4j, avoid System.out)
- [ ] Magic values extracted to constants or enums
