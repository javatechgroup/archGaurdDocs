# AG016: Unpaginated REST Collections

| Attribute | Details |
| :--- | :--- |
| **Rule ID** | `AG016` |
| **Name** | Unpaginated REST Collections |
| **Default Severity** | `WARNING` |
| **Category** | Memory Safety & API Scaling |
| **Analysis Engine** | AST Source Analysis (JavaParser) |
| **Core Classes** | `AG016UnpaginatedCollectionRule, RestApiInspector` |

---

## Executive Summary & Architectural Rationale

Returning unbounded collections from HTTP GET endpoints is an enterprise stability risk:
1. **OutOfMemoryError Outages:** As database tables grow from hundreds of rows to millions, an unpaginated `GET /api/items` endpoint attempts to load all records into JVM heap memory, crashing the application.
2. **Massive Network Latency:** Serializing megabytes of JSON over HTTP stalls client mobile apps and exhausts network bandwidth.
3. **Database Connection Pool Exhaustion:** Full table scans tie up JDBC connections in HikariCP for prolonged durations, causing thread starvation.

---

## How It Is Implemented

AST analysis inspects all `@GetMapping` or `@RequestMapping(method = GET)` methods in `@RestController` classes:
1. Checks return type: matches `List<T>`, `Set<T>`, `ResponseEntity<List<T>>`, or `ResponseEntity<Set<T>>`.
2. Checks method parameters for pagination controllers: `Pageable` or `PageRequest`.
3. If pagination parameters are absent, records `UnpaginatedCollectionFinding`.

---

### False Positive (FP) Handling & Mitigations
1. **Methods Accepting `Pageable`:**
   - If the endpoint accepts `Pageable pageable` or `PageRequest`, the developer has provided pagination control, avoiding false alerts even if returning a list.
2. **Spring Data `Page<T>` and `Slice<T>`:**
   - Methods returning `Page<UserDto>` or `Slice<UserDto>` are paginated by definition and are completely compliant.
3. **Single Object DTOs:**
   - Returning single records, DTOs, or summaries does not trigger AG016.

---

### False Negative (FN) Handling & Defense-in-Depth
1. **ResponseEntity Wrappers:**
   - Accurately inspects generic return types wrapped in `ResponseEntity<List<T>>`.

---

## Code Examples

### Non-Compliant (Violating Anti-Pattern)

```java
// VIOLATION: Unpaginated List return type in GET endpoint
@RestController
@RequestMapping("/api/customers")
public class CustomerController {

    @GetMapping
    public List<CustomerDto> getAllCustomers() { // Unbounded query! OOM risk.
        return customerService.findAll();
    }
}
```

### Compliant (Refactored Best Practice)

```java
// COMPLIANT: Returning Spring Data Page<T> with Pageable
@RestController
@RequestMapping("/api/customers")
public class CustomerController {

    @GetMapping
    public Page<CustomerDto> getCustomers(Pageable pageable) { // Safe, bounded pagination
        return customerService.findPage(pageable);
    }
}
```

### Legitimate Edge Cases Handled Without Alerting

```java
// COMPLIANT: Method accepting Pageable
@RestController
public class OrderController {
    @GetMapping("/recent")
    public List<OrderDto> getRecent(Pageable pageable) { // Parameter provides bounds: permitted
        return orderService.findRecent(pageable);
    }
}
```

---

## Configuration & Tuning

In `archguard.yml`, you can customize rule severity, exclude specific classes, or adjust thresholds:

```yaml
rules:
  AG016:
    enabled: true
    severity: WARNING # Options: ERROR, WARNING, INFO
    exclude:
      - "**/legacy/**"
      - "**/*Test.java"
```

### In-Code Suppression
You can suppress this rule directly in source code using either standard Java annotations or ArchGuard annotations:

```java
// Suppress on a specific method
@SuppressArchitecture("AG016")
public void myMethod() {
    // ...
}

// Or using standard Java @SuppressWarnings
@SuppressWarnings("archguard:AG016")
public void myMethod() {
    // ...
}
```
