# AG021: Spring Proxy Self-Invocation

| Attribute | Details |
| :--- | :--- |
| **Rule ID** | `AG021` |
| **Name** | Spring Proxy Self-Invocation |
| **Default Severity** | `ERROR` |
| **Category** | Spring Runtime Traps |
| **Analysis Engine** | AST Source Analysis (JavaParser) |
| **Core Classes** | `AG021SelfInvocationRule, SpringProxySelfInvocationInspector` |

---

## Executive Summary & Architectural Rationale

Spring AOP is proxy-based: Spring wraps beans in dynamic CGLIB or JDK proxies that intercept incoming method calls to apply transaction management, asynchronous execution, security checks, and caching.
1. **Silent Transaction Bypass:** Calling `this.updateBalance()` from within the same class invokes the raw target instance method directly, bypassing the Spring proxy. Any `@Transactional` annotation on `updateBalance()` is silently ignored, running without a database transaction!
2. **Broken Async Scheduling:** Calling `this.sendEmailAsync()` internally executes synchronously on the caller thread instead of using the background task executor.
3. **Missing Cache Hits:** Calling `this.getCachedData()` skips Spring CacheManager lookups.

---

## How It Is Implemented

AST analysis inspects Spring bean components (`@Service`, `@Component`, etc.):
1. **Proxy Method Indexing:** Scans all methods for Spring proxy annotations:
   - `@Transactional`, `@Async`, `@Cacheable`, `@CachePut`, `@CacheEvict`, `@Retryable`, `@Scheduled`.
   - Indexes method signature including parameter count arity (`name#paramCount`).
2. **Local Method Call Inspection:**
   - Traverses method calls within the bean.
   - Filters for calls where scope is empty or `this`.
   - Checks if target method matches a proxy-annotated signature.
   - Excludes recursive calls to the exact same method.
3. Emits `SelfInvocationFinding` with caller and target method details.

---

### False Positive (FP) Handling & Mitigations
1. **Method Overload Arity Matching:**
   - If a class defines `process()` (unannotated) and `process(String id)` (annotated with `@Transactional`), calling `this.process()` does **not** trigger a violation because the parameter count signature (`process#0` vs `process#1`) distinguishes the overload.
2. **Calls on Injected Collaborators:**
   - If `OrderService` injects `PaymentService` and calls `paymentService.process()`, the scope is `paymentService` (not `this`). These cross-bean calls pass through Spring proxies and are compliant.
3. **Recursive Invocations:**
   - A method calling itself recursively with the same signature is excluded.

---

### False Negative (FN) Handling & Defense-in-Depth
1. **Unqualified Method Calls:**
   - In Java, calling `method()` without `this.` is semantically identical to `this.method()`. The inspector checks both `call.getScope().isEmpty()` and `call.getScope().get().toString().equals("this")`.

---

## Code Examples

### Non-Compliant (Violating Anti-Pattern)

```java
// VIOLATION: Internal call silently bypasses @Transactional proxy
@Service
public class OrderService {

    public void processOrder(Order order) {
        // Direct internal call to sibling method!
        this.saveOrderWithTransaction(order); // @Transactional is completely ignored!
    }

    @Transactional
    public void saveOrderWithTransaction(Order order) {
        orderRepository.save(order);
    }
}
```

### Compliant (Refactored Best Practice)

```java
// COMPLIANT: Extracted to dedicated collaborator or TransactionTemplate
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderTxService orderTxService; // Injected proxy collaborator

    public void processOrder(Order order) {
        orderTxService.saveOrderWithTransaction(order); // Passes through Spring proxy!
    }
}

// ALSO COMPLIANT: Using TransactionTemplate
@Service
@RequiredArgsConstructor
public class OrderService {
    private final TransactionTemplate transactionTemplate;

    public void processOrder(Order order) {
        transactionTemplate.executeWithoutResult(status -> {
            orderRepository.save(order);
        });
    }
}
```

### Legitimate Edge Cases Handled Without Alerting

```java
// COMPLIANT: Recursive method calling itself
@Service
public class TreeService {
    @Transactional
    public void traverse(Node node) {
        if (node == null) return;
        traverse(node.getLeft()); // Direct recursion: excluded from proxy bypass alerts
        traverse(node.getRight());
    }
}
```

---

## Configuration & Tuning

In `archguard.yml`, you can customize rule severity, exclude specific classes, or adjust thresholds:

```yaml
rules:
  AG021:
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
@SuppressArchitecture("AG021")
public void myMethod() {
    // ...
}

// Or using standard Java @SuppressWarnings
@SuppressWarnings("archguard:AG021")
public void myMethod() {
    // ...
}
```
