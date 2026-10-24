# AG028: ThreadLocal Memory Leak

| Attribute | Details |
| :--- | :--- |
| **Rule ID** | `AG028` |
| **Name** | ThreadLocal Memory Leak |
| **Default Severity** | `ERROR` |
| **Category** | Memory Safety & Concurrency |
| **Analysis Engine** | AST Source Analysis (Lifecycle & Finally Inspection) |
| **Core Classes** | `AG028ThreadLocalLeakRule, ConcurrencyInspector` |

---

## Executive Summary & Architectural Rationale

`ThreadLocal` memory leaks are among the most dangerous issues in pooled server environments:
1. **PermGen/Metaspace & Heap Memory Leaks:** Web servers (Tomcat, Jetty, Undertow) use pooled worker threads that stay alive for days or weeks. Objects stored in `ThreadLocal` remain referenced by the thread's `ThreadLocalMap` indefinitely if not explicitly removed, leading to massive heap memory leaks.
2. **Cross-Request Security Contamination:** If Thread 1 handles Request A for User 123 and fails to call `remove()`, when Thread 1 is reused by the thread pool for Request B from User 456, Request B inadvertently reads User 123's security context or tenant ID!
3. **ClassLoader Pinning:** Storing application classes in `ThreadLocal` prevents ClassLoader unloading during hot-reloads.

---

## How It Is Implemented

AST analysis inspects `MethodDeclaration` nodes for `ThreadLocal.set()` calls:
1. Identifies `call.getNameAsString().equals("set")` where target scope is a `ThreadLocal`.
2. **Finally Block Verification:**
   - Searches enclosing method's `TryStmt` nodes.
   - Verifies if `finallyBlock` contains `threadLocal.remove()`.
3. **Inter-Method Lifecycle Verification:**
   - In lifecycle handler components (e.g. `HandlerInterceptor`), checks if a sibling method in the same class (such as `afterCompletion()`) invokes `remove()`.
4. If neither finally-remove nor companion-remove exists, records `ThreadLocalLeakFinding`.

---

### False Positive (FP) Handling & Mitigations
1. **Guaranteed `finally { threadLocal.remove(); }`:**
   - If `remove()` is invoked inside an enclosing `finally` block, execution is guaranteed even if exceptions occur. AG028 recognizes this compliant pattern.
2. **Spring `HandlerInterceptor` Lifecycle Pairing:**
   - Web interceptors commonly set tenant context in `preHandle()` and clean it in `afterCompletion()`. The inspector scans sibling methods in the class to recognize lifecycle cleanup and avoid false alarms.
3. **Java 21 `ScopedValue` Alternatives:**
   - Using modern Java 21 `ScopedValue.where(...).run(...)` does not use `ThreadLocal` and is fully compliant.

---

### False Negative (FN) Handling & Defense-in-Depth
1. **Removing in Normal Control Flow Without `finally`:**
   - Calling `threadLocal.remove()` at the very bottom of a method outside `finally` is caught because an unexpected exception will skip the remove call and leak memory.
2. **Scope Name Resolution:**
   - Matches field names and local variable names of `ThreadLocal` types.

---

## Code Examples

### Non-Compliant (Violating Anti-Pattern)

```java
// VIOLATION: ThreadLocal.set() without guaranteed finally { remove(); }
@Component
public class TenantInterceptor {

    private static final ThreadLocal<String> TENANT_HOLDER = new ThreadLocal<>();

    public void processTenant(String tenantId) {
        TENANT_HOLDER.set(tenantId); // If processing throws, tenant leaks into pooled thread!
        doBusinessLogic();
        TENANT_HOLDER.remove(); // Bypassed on exception!
    }
}
```

### Compliant (Refactored Best Practice)

```java
// COMPLIANT: Guaranteed cleanup in finally block
@Component
public class TenantInterceptor {

    private static final ThreadLocal<String> TENANT_HOLDER = new ThreadLocal<>();

    public void processTenant(String tenantId) {
        TENANT_HOLDER.set(tenantId);
        try {
            doBusinessLogic();
        } finally {
            TENANT_HOLDER.remove(); // Always cleaned up
        }
    }
}

// ALSO COMPLIANT: HandlerInterceptor preHandle / afterCompletion pairing
public class WebTenantInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(...) {
        TENANT_HOLDER.set(tenantId);
        return true;
    }

    @Override
    public void afterCompletion(...) {
        TENANT_HOLDER.remove(); // Recognized lifecycle cleanup
    }
}
```

### Legitimate Edge Cases Handled Without Alerting

```java
// COMPLIANT: Modern Java 21 ScopedValue (immune to ThreadLocal leaks)
public static final ScopedValue<String> TENANT_ID = ScopedValue.newInstance();

ScopedValue.where(TENANT_ID, "tenant-1").run(() -> {
    // Automatically cleaned up when lambda finishes
});
```

---

## Configuration & Tuning

In `archguard.yml`, you can customize rule severity, exclude specific classes, or adjust thresholds:

```yaml
rules:
  AG028:
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
@SuppressArchitecture("AG028")
public void myMethod() {
    // ...
}

// Or using standard Java @SuppressWarnings
@SuppressWarnings("archguard:AG028")
public void myMethod() {
    // ...
}
```
