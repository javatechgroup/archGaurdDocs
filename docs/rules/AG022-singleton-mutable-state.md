# AG022: Singleton Mutable Instance State

| Attribute | Details |
| :--- | :--- |
| **Rule ID** | `AG022` |
| **Name** | Singleton Mutable Instance State |
| **Default Severity** | `ERROR` |
| **Category** | Concurrency & Thread Safety |
| **Analysis Engine** | AST Source Analysis (JavaParser) |
| **Core Classes** | `AG022SingletonMutableStateRule, InjectionAndStateInspector, AstUtils` |

---

## Executive Summary & Architectural Rationale

Spring beans (`@Service`, `@Component`, `@RestController`) are singletons by default. A single bean instance handles hundreds of concurrent HTTP requests across worker threads:
1. **SimpleDateFormat Race Conditions:** `SimpleDateFormat` maintains internal calendar state during parsing and formatting. When shared concurrently across request threads, it corrupts date values and throws random `NumberFormatException` or `ArrayIndexOutOfBoundsException`.
2. **Cross-User Data Leakage:** Mutable instance variables (e.g. `private Long currentUserId;`) hold state across requests. If Thread A sets `currentUserId = 123` and is preempted, Thread B sets `currentUserId = 456`, Thread A resumes and operates on the wrong user's data!
3. **Data Race Inconsistencies:** Mutable collections (`ArrayList`, `HashMap`) mutated without synchronization corrupt internal hash table buckets, leading to infinite loops in `HashMap.get()`.

---

## How It Is Implemented

AST analysis inspects instance fields in Spring singleton beans:
1. Filters out static fields (handled separately by AG010).
2. **Scope Verification:**
   - Excludes beans explicitly annotated with `@RequestScope`, `@SessionScope`, or `@Scope("prototype")`.
   - Excludes configuration binding beans (`@ConfigurationProperties`).
3. **Field Categorization:**
   - Excludes injected dependencies (`@Autowired`, `@Inject`, `@Value`, `@PersistenceContext`).
   - Excludes loggers (`Logger`).
   - Flags non-thread-safe formatters: `SimpleDateFormat`, `DateFormat`, `DecimalFormat`, `Calendar`.
   - Flags non-final mutable collections (`List`, `Map`, `Set`).
   - Flags non-final primitive and primitive wrapper fields (`int`, `boolean`, `String`).

---

### False Positive (FP) Handling & Mitigations
1. **Injected Collaborators & Properties:**
   - Fields carrying `@Autowired`, `@Value("${...}")`, or `@PersistenceContext` are framework dependencies, not mutable request state, and are excluded.
2. **Request-Scoped & Prototype Beans:**
   - Beans annotated with `@RequestScope`, `@SessionScope`, or `@Scope("prototype")` are instantiated per-request or per-call and are safely exempted.
3. **Final Immutable Fields:**
   - Fields marked `final` and initialized to immutable values are compliant.
4. **Configuration Properties:**
   - Classes annotated with `@ConfigurationProperties` are designed to hold mutable configuration state and are exempted.
5. **Domain Models and DTOs Named `Component`:**
   - Certain business models or UI/payload classes may be named `Component` (e.g., `com.company.dto.Component` representing a UI component or an entity representing a hardware component). When located in `dto`, `model`, `entity`, `payload`, or `domain` packages and not annotated with Spring's `@Component` or `@Service`, they are classified as data models/POJOs and are exempted from singleton thread-safety inspection.

---

### False Negative (FN) Handling & Defense-in-Depth
1. **Implicit Bean Suffixes:**
   - Inspects classes ending in `Service`, `Component`, `Controller` even if Spring stereotype annotation is inherited or implied.
2. **Unsafe Date Formatters:**
   - Flags `SimpleDateFormat` even if marked `final`, because `SimpleDateFormat` is internally mutable and fundamentally not thread-safe!

---

## Code Examples

### Non-Compliant (Violating Anti-Pattern)

```java
// VIOLATION: Mutable state and non-thread-safe SimpleDateFormat in singleton
@Service
public class InvoiceService {

    private SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd"); // RACE HAZARD!

    private String currentInvoiceId; // Cross-request data corruption!
}
```

### Compliant (Refactored Best Practice)

```java
// COMPLIANT: Thread-safe DateTimeFormatter and local method variables
@Service
public class InvoiceService {

    private static final DateTimeFormatter DATE_FORMATTER = DateTimeFormatter.ISO_LOCAL_DATE; // Thread-safe

    public void processInvoice(String invoiceId) {
        // State kept as local variable on thread stack
        LocalDate date = LocalDate.now();
        String formatted = date.format(DATE_FORMATTER);
    }
}
```

### Legitimate Edge Cases Handled Without Alerting

```java
// COMPLIANT: Explicitly request-scoped bean
@Component
@RequestScope // Fresh instance per HTTP request: permitted
public class UserSessionContext {
    private String userId; // Safe in request scope
}

// COMPLIANT: Data model / DTO named Component (not a Spring singleton bean)
package com.company.dto.whatsapp;

public class Component { // Permitted: DTO / POJO data model
    private String type;
    private List<Parameter> parameters;
}
```

---

## Configuration & Tuning

In `archguard.yml`, you can customize rule severity, exclude specific classes, or adjust thresholds:

```yaml
rules:
  AG022:
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
@SuppressArchitecture("AG022")
public void myMethod() {
    // ...
}

// Or using standard Java @SuppressWarnings
@SuppressWarnings("archguard:AG022")
public void myMethod() {
    // ...
}
```
