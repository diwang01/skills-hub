# Architecture Review Guide

Architecture design review guide to help evaluate code architecture and design quality.

## SOLID Principles Checklist

### S - Single Responsibility Principle (SRP)

**Check Points:**
- Does this class/module have only one reason to change?
- Do all methods in the class serve the same purpose?
- Can you describe this class to a non-technical person in one sentence?

**Warning Signs in Code Review:**
```
- Class name contains "And", "Manager", "Handler", "Processor" or other generic terms
- A class exceeds 200-300 lines of code
- Class has more than 5-7 public methods
- Different methods operate on completely different data
```

**Review Questions:**
- "What responsibilities does this class have? Can it be split?"
- "If requirement X changes, which methods need modification? What about requirement Y?"

### O - Open/Closed Principle (OCP)

**Check Points:**
- Do you need to modify existing code when adding new features?
- Can new behavior be added through extension (inheritance, composition)?
- Are there many if/else or switch statements handling different types?

**Warning Signs in Code Review:**
```
- switch/if-else chains handling different types
- Adding new features requires modifying core classes
- Type checks (instanceof) scattered throughout code
```

**Review Questions:**
- "If we need to add a new type X, how many files need modification?"
- "Will this switch statement grow with new types?"

### L - Liskov Substitution Principle (LSP)

**Check Points:**
- Can subclasses completely replace parent classes?
- Do subclasses change expected behavior of parent class methods?
- Do subclasses throw exceptions not declared by parent class?

**Warning Signs in Code Review:**
```
- Explicit type casting
- Subclass methods throwing NotImplementedException
- Empty implementations or just return statements in subclass methods
- Places using base class need to check specific types
```

**Review Questions:**
- "If we replace parent class with subclass, does caller code need modification?"
- "Does this method's behavior in subclass conform to parent class contract?"

### I - Interface Segregation Principle (ISP)

**Check Points:**
- Are interfaces small and focused enough?
- Are implementing classes forced to implement unneeded methods?
- Do clients depend on methods they don't use?

**Warning Signs in Code Review:**
```
- Interface has more than 5-7 methods
- Implementing classes have empty methods or throw NotImplementedException
- Interface names too broad (IManager, IService)
- Different clients only use part of interface methods
```

**Review Questions:**
- "Are all methods of this interface used by every implementing class?"
- "Can this large interface be split into smaller specialized interfaces?"

### D - Dependency Inversion Principle (DIP)

**Check Points:**
- Do high-level modules depend on abstractions rather than concrete implementations?
- Is dependency injection used instead of direct object instantiation?
- Are abstractions defined by high-level modules rather than low-level modules?

**Warning Signs in Code Review:**
```
- High-level modules directly new low-level module concrete classes
- Importing concrete implementation classes instead of interfaces/abstract classes
- Configuration and connection strings hardcoded in business logic
- Difficulty writing unit tests for a class
```

**Review Questions:**
- "Can this class's dependencies be mocked in tests?"
- "If we need to change database/API implementation, how many places need modification?"

---

## Architecture Anti-Pattern Identification

### Fatal Anti-Patterns

| Anti-Pattern | Warning Signs | Impact |
|--------------|---------------|--------|
| **Big Ball of Mud** | No clear module boundaries, any code can call any other code | Hard to understand, modify, and test |
| **God Object** | Single class takes too many responsibilities, knows too much, does too much | High coupling, hard to reuse and test |
| **Spaghetti Code** | Chaotic control flow, goto or deep nesting, hard to trace execution path | Hard to understand and maintain |
| **Lava Flow** | Ancient code no one dares to touch, lacking documentation and tests | Technical debt accumulation |

### Design Anti-Patterns

| Anti-Pattern | Warning Signs | Suggestion |
|--------------|---------------|------------|
| **Golden Hammer** | Using same technology/pattern for all problems | Choose solution based on problem |
| **Over-Engineering** | Complex solution for simple problem, design pattern abuse | YAGNI principle, simple first |
| **Boat Anchor** | Unused code written for "future needs" | Delete unused code, write when needed |
| **Copy-Paste Programming** | Same logic appears in multiple places | Extract common method or module |

### Review Comments

```markdown
[blocking] "This class has 2000 lines of code, suggest splitting into focused classes"
[important] "This logic is duplicated in 3 places, consider extracting to common method?"
[suggestion] "This switch statement could be replaced with Strategy pattern for easier extension"
```

---

## Coupling & Cohesion Assessment

### Coupling Types (Good to Bad)

| Type | Description | Example |
|------|-------------|---------|
| **Message Coupling** | Pass data through parameters | `calculate(price, quantity)` |
| **Data Coupling** | Share simple data structures | `processOrder(orderDTO)` |
| **Stamp Coupling** | Share complex data but only use part | Pass entire User object but only use name |
| **Control Coupling** | Pass control flags affecting behavior | `process(data, isAdmin=true)` |
| **Common Coupling** | Share global variables | Multiple modules read/write same global state |
| **Content Coupling** | Directly access another module's internals | Directly manipulate another class's private properties |

### Cohesion Types (Good to Bad)

| Type | Description | Quality |
|------|-------------|---------|
| **Functional Cohesion** | All elements complete a single task | Best |
| **Sequential Cohesion** | Output serves as input for next step | Good |
| **Communicational Cohesion** | Operate on same data | Acceptable |
| **Temporal Cohesion** | Tasks executed at same time | Poor |
| **Logical Cohesion** | Logically related but functionally different | Bad |
| **Coincidental Cohesion** | No apparent relationship | Worst |

### Metric References

```yaml
Coupling Metrics:
  CBO (Coupling Between Objects):
    Good: < 5
    Warning: 5-10
    Danger: > 10

  Ce (Efferent Coupling):
    Description: How many external classes it depends on
    Good: < 7

  Ca (Afferent Coupling):
    Description: How many classes depend on it
    High value means: Modification impact is large, needs stability

Cohesion Metrics:
  LCOM4 (Lack of Cohesion of Methods):
    1: Single responsibility - Good
    2-3: May need splitting - Warning
    >3: Should split - Bad
```

### Review Questions

- "How many other modules does this module depend on? Can we reduce it?"
- "How many other places will be affected by modifying this class?"
- "Do all methods of this class operate on the same data?"

---

## Layered Architecture Review

### Clean Architecture Layer Checks

```
+-------------------------------------+
|         Frameworks & Drivers        | <- Outermost: Web, DB, UI
+------------------------------------|
|         Interface Adapters          | <- Controllers, Gateways, Presenters
+-------------------------------------+
|          Application Layer          | <- Use Cases, Application Services
+-------------------------------------+
|            Domain Layer             | <- Entities, Domain Services
+-------------------------------------+
          ^ Dependencies only point inward ^
```

### Dependency Rule Checks

**Core Rule: Source code dependencies can only point inward**

```java
// Bad: Domain layer depends on Infrastructure
// domain/User.java
import com.example.infrastructure.MySQLConnection;

// Good: Domain layer defines interface, Infrastructure implements
// domain/UserRepository.java (interface)
public interface UserRepository {
    Optional<User> findById(String id);
}

// infrastructure/MySQLUserRepository.java (implementation)
@Repository
public class MySQLUserRepository implements UserRepository {
    public Optional<User> findById(String id) { /* ... */ }
}
```

### Review Checklist

**Layer Boundary Checks:**
- [ ] Does Domain layer have external dependencies (database, HTTP, file system)?
- [ ] Does Application layer directly operate database or call external APIs?
- [ ] Does Controller contain business logic?
- [ ] Are there cross-layer calls (UI directly calling Repository)?

**Separation of Concerns Checks:**
- [ ] Is business logic separated from presentation logic?
- [ ] Is data access encapsulated in dedicated layer?
- [ ] Is configuration and environment-related code centrally managed?

### Review Comments

```markdown
[blocking] "Domain entity directly imports database connection, violates dependency rule"
[important] "Controller contains business calculation logic, suggest moving to Service layer"
[suggestion] "Consider using dependency injection to decouple these components"
```

---

## Design Pattern Usage Assessment

### When to Use Design Patterns

| Pattern | Suitable Scenarios | Unsuitable Scenarios |
|---------|-------------------|---------------------|
| **Factory** | Need to create different types of objects, type determined at runtime | Only one type, or type is fixed |
| **Strategy** | Algorithm needs runtime switching, multiple interchangeable behaviors | Only one algorithm, or algorithm won't change |
| **Observer** | One-to-many dependency, state changes need to notify multiple objects | Simple direct calls satisfy needs |
| **Singleton** | Truly need globally unique instance, like configuration management | Objects that can be passed via dependency injection |
| **Decorator** | Need to dynamically add responsibilities, avoid inheritance explosion | Responsibilities fixed, no dynamic composition needed |

### Over-Engineering Warning Signs

```
Patternitis Warning Signs:

1. Simple if/else replaced with Strategy pattern + Factory + Registry
2. Interface with only one implementation
3. Abstraction layers added for "future needs"
4. Code lines significantly increased due to pattern application
5. New developers need long time to understand code structure
```

### Review Principles

```markdown
Correct pattern usage:
- Solves actual extensibility problems
- Code becomes easier to understand and test
- Adding new features becomes simpler

Pattern overuse:
- Using pattern for pattern's sake
- Added unnecessary complexity
- Violates YAGNI principle
```

### Review Questions

- "What specific problem does this pattern solve?"
- "If we don't use this pattern, what problems would the code have?"
- "Is the value of this abstraction layer greater than its complexity?"

---

## Extensibility Assessment

### Extensibility Checklist

**Feature Extensibility:**
- [ ] Does adding new features require modifying core code?
- [ ] Are extension points provided (hooks, plugins, events)?
- [ ] Is configuration externalized (config files, environment variables)?

**Data Extensibility:**
- [ ] Does data model support adding new fields?
- [ ] Is data volume growth scenario considered?
- [ ] Do queries have appropriate indexes?

**Load Extensibility:**
- [ ] Can it scale horizontally (add more instances)?
- [ ] Are there state dependencies (session, local cache)?
- [ ] Does database connection use connection pool?

### Extension Point Design Checks

```java
// Good: Extension design using events/hooks
public class OrderService {
    private final OrderHooks hooks;

    public Order createOrder(Order order) {
        hooks.beforeCreate(order);
        Order result = save(order);
        hooks.afterCreate(result);
        return result;
    }
}

// Bad: Extension design with hardcoded behavior
public class OrderService {
    public Order createOrder(Order order) {
        sendEmail(order);        // Hardcoded
        updateInventory(order);  // Hardcoded
        notifyWarehouse(order);  // Hardcoded
        return save(order);
    }
}
```

### Review Questions

```markdown
[suggestion] "If we need to support new payment methods in future, is this design easy to extend?"
[important] "Logic here is hardcoded, consider using configuration or Strategy pattern?"
[learning] "Event-driven architecture can make this feature easier to extend"
```

---

## Code Structure Best Practices

### Directory Organization

**Organize by Feature/Domain (Recommended):**
```
src/
├── user/
│   ├── User.java           (Entity)
│   ├── UserService.java    (Service)
│   ├── UserRepository.java (Data Access)
│   └── UserController.java (API)
├── order/
│   ├── Order.java
│   ├── OrderService.java
│   └── ...
└── shared/
    ├── utils/
    └── types/
```

**Organize by Technical Layer (Not Recommended):**
```
src/
├── controllers/     <- Different domains mixed together
│   ├── UserController.java
│   └── OrderController.java
├── services/
├── repositories/
└── models/
```

### Naming Convention Checks

| Type | Convention | Example |
|------|-----------|---------|
| Class Name | PascalCase, Noun | `UserService`, `OrderRepository` |
| Method Name | camelCase, Verb | `createUser`, `findOrderById` |
| Interface Name | I prefix or none | `IUserService` or `UserService` |
| Constants | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT` |
| Private Fields | underscore prefix or none | `_cache` or `cache` |

### File Size Guidelines

```yaml
Recommended Limits:
  Single File: < 300 lines
  Single Function: < 50 lines
  Single Class: < 200 lines
  Function Parameters: < 4
  Nesting Depth: < 4 levels

When Limits Exceeded:
  - Consider splitting into smaller units
  - Use composition over inheritance
  - Extract helper functions or classes
```

### Review Comments

```markdown
[nit] "This 500-line file could be split by responsibility"
[important] "Suggest organizing directories by feature domain rather than technical layer"
[suggestion] "Function name `process` is not clear enough, consider renaming to `calculateOrderTotal`?"
```

---

## Quick Reference Checklist

### 5-Minute Architecture Review

```markdown
□ Is dependency direction correct? (outer depends on inner)
□ Are there circular dependencies?
□ Is core business logic decoupled from framework/UI/database?
□ Does it follow SOLID principles?
□ Are there obvious anti-patterns?
```

### Red Flags (Must Address)

```markdown
- God Object: Single class exceeds 1000 lines
- Circular Dependency: A -> B -> C -> A
- Domain layer contains framework dependencies
- Hardcoded configuration and secrets
- External service calls without interfaces
```

### Yellow Flags (Should Address)

```markdown
- Class coupling (CBO) > 10
- Method parameters exceed 5
- Nesting depth exceeds 4 levels
- Duplicate code blocks > 10 lines
- Interface with only one implementation
```

---

## Tool Recommendations

| Tool | Purpose | Language Support |
|------|---------|------------------|
| **SonarQube** | Code quality, coupling analysis | Multi-language |
| **JDepend** | Package dependency analysis | Java |
| **ArchUnit** | Architecture rules testing | Java |
| **SpotBugs** | Bug pattern detection | Java |
| **Checkstyle** | Code style checking | Java |
| **PMD** | Code quality rules | Java |
