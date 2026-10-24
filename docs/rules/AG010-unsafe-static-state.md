# AG010: Unsafe Mutable Static State

| Attribute | Details |
| :--- | :--- |
| **Rule ID** | `AG010` |
| **Name** | Unsafe Mutable Static State |
| **Default Severity** | `ERROR` |
| **Category** | Concurrency & Thread Safety |
| **Analysis Engine** | AST Source Analysis (JavaParser) |
| **Core Classes** | `AG010UnsafeMutableStaticStateRule, InjectionAndStateInspector, AstUtils` |

---

## Executive Summary & Architectural Rationale

In Spring Boot applications, beans are singletons by default, shared across all Tomcat or Undertow request threads:
1. **Multi-Threaded Data Races:** Non-final static fields or non-thread-safe mutable collections (`static List`, `static Map`, `static HashMap`) accessed by concurrent HTTP threads cause race conditions, data overwrites, and `ConcurrentModificationException`.
2. **Memory Leaks:** Static collections retain object references across the lifetime of the JVM ClassLoader, preventing garbage collection and causing OutOfMemoryError.
3. **Cross-Tenant Contamination:** In multi-tenant environments, static fields inadvertently leak user data between concurrent sessions.

---

## How It Is Implemented

AST analysis inspects fields in Spring bean declarations (`@Component`, `@Service`, `@RestController`, `@Controller`, `@Repository`, `@Configuration`):
1. Filters for `field.isStatic()`.
2. **Non-Final Static State:** If `!field.isFinal()`, flags immediately as a race hazard.
3. **Mutable Static Collection:**
   - If `field.isFinal()`, checks if declared type is a non-thread-safe mutable collection (`List`, `Map`, `Set`, `ArrayList`, `HashMap`, etc.).
   - Checks the initialization expression using `AstUtils.isImmutableCollectionInitializer()`.
   - Checks if the collection is thread-safe (`AstUtils.isThreadSafeConcurrentType()`, `AstUtils.isThreadSafeInitializer()`).
   - If not immutable and not a concurrent/thread-safe collection, flags as an unsafe mutable collection shared across threads.

---

### False Positive (FP) Handling & Mitigations
1. **Primitive & String Constants:**
   - `public static final String API_VERSION = "v1";` and `private static final int MAX_RETRIES = 3;` are immutable constants and are completely ignored.
2. **Modern Immutable Collection Initializers:**
   - `AstUtils.isImmutableCollectionExpression()` recognizes:
     - `List.of(...)`, `Set.of(...)`, `Map.of(...)`, `Map.ofEntries(...)`
     - `List.copyOf(...)`, `Set.copyOf(...)`, `Map.copyOf(...)`
     - `Collections.unmodifiableList(...)`, `Collections.emptyMap(...)`, `Collections.singleton(...)`
     - Java 16+ `Stream.toList()` and `Collectors.toUnmodifiableList()`
     - Guava `ImmutableList.builder()...build()`
   - These are guaranteed immutable and are **not** flagged.
3. **Non-Spring Utility Classes:**
   - Static state in standalone utility classes without Spring stereotypes is exempted.
4. **Thread-Safe Concurrent Collections (`ConcurrentHashMap`, `CopyOnWriteArrayList`, etc.):**
   - Thread-safe concurrent collections (`ConcurrentHashMap`, `ConcurrentMap`, `CopyOnWriteArrayList`, `CopyOnWriteArraySet`, `ConcurrentLinkedQueue`, `BlockingQueue`, `Atomic*`) are designed for safe multi-threaded concurrent access and are exempted from unsafe mutable state inspection.

---

### False Negative (FN) Handling & Defense-in-Depth
1. **`static final` Mutable Collections:**
   - Naive analyzers only check `!field.isFinal()` and miss `public static final List<String> CACHE = new ArrayList<>();`. AG010 inspects collection types even when `final`.
2. **Nested Expressions:**
   - Unpacks cast expressions `(List<String>) List.of(...)` and parenthesized expressions before evaluating immutability.

---

## Code Examples

### Non-Compliant (Violating Anti-Pattern)

```java
// VIOLATION: Static mutable field and non-thread-safe HashMap in singleton bean
@Service
public class OrderMetricsService {

    public static int requestCount = 0; // Data race across HTTP threads!

    public static final Map<String, Order> orderCache = new HashMap<>(); // Not thread-safe!
}
```

### Compliant (Refactored Best Practice)

```java
// COMPLIANT: Atomic primitives, immutable collections, or ConcurrentHashMap
@Service
public class OrderMetricsService {

    private final AtomicInteger requestCount = new AtomicInteger(0); // Thread-safe atomic

    private static final List<String> SUPPORTED_CURRENCIES = List.of("USD", "EUR", "GBP"); // Immutable

    private static final ConcurrentHashMap<String, Order> ORDER_CACHE = new ConcurrentHashMap<>(); // Concurrent safe
}
```

### Legitimate Edge Cases Handled Without Alerting

```java
// COMPLIANT: Collections.unmodifiableMap initializer
@Service
public class RateLimitService {
    private static final Map<String, Integer> LIMITS = Collections.unmodifiableMap(Map.of("GOLD", 100));
}

// COMPLIANT: ConcurrentHashMap in-memory cache in service
@Service
public class ConstCacheService {
    private static final ConcurrentHashMap<String, WaConst> CONST_CACHE = new ConcurrentHashMap<>();
}
```

---

## Configuration & Tuning

In `archguard.yml`, you can customize rule severity, exclude specific classes, or adjust thresholds:

```yaml
rules:
  AG010:
    enabled: true
    severity: ERROR # Options: ERROR, WARNING, INFO
    exclude:
      - "**/legacy/**"
      - "**/*Test.java"
```

### In-Code Suppression
You can suppress this rule directly in source code using either standard Java annotations or ArchGuard annotations:

```java
// Suppress on a specific method
@SuppressArchitecture("AG010")
public void myMethod() {
    // ...
}

// Or using standard Java @SuppressWarnings
@SuppressWarnings("archguard:AG010")
public void myMethod() {
    // ...
}
```
