# AG008: Swallowed Exceptions

| Attribute | Details |
| :--- | :--- |
| **Rule ID** | `AG008` |
| **Name** | Swallowed Exceptions |
| **Default Severity** | `WARNING` |
| **Category** | Error Handling & Reliability |
| **Analysis Engine** | AST Source Analysis (JavaParser) |
| **Core Classes** | `AG008SwallowedExceptionRule, ResourceAndSafetyInspector` |

---

## Executive Summary & Architectural Rationale

Swallowed exceptions represent one of the most perilous software anti-patterns:
1. **Silent Production Failures:** When an exception is caught and discarded, critical operations fail silently without alerting on-call engineers or leaving any audit trace.
2. **Corrupted State & Broken Transactions:** Catching an exception without logging or rethrowing prevents Spring from rolling back the current transaction, committing corrupted partial database writes.
3. **Lost Root Cause:** Without logging the stack trace (`log.error("...", e)`), root causes (network drops, deadlocks, bad input) vanish forever, turning production debugging into pure guesswork.

---

## How It Is Implemented

AST analysis inspects all `CatchClause` nodes:
1. **Catch Parameter Inspection:**
   - Inspects the catch parameter variable name (e.g. `catch (Exception e)`).
   - If parameter is named `ignored`, `expected`, `ignore`, or `expect`, the developer intentionally opted out per standard Java conventions.
2. **Body Statement Traversal:**
   - If `catch` block is empty, it is flagged immediately.
   - Searches for `ThrowStmt` (`throw ...`). If present, exception is propagated.
   - Searches for `MethodCallExpr` matching logging methods: `log.*`, `logger.*`, `error()`, `warn()`, `info()`, `printStackTrace()`.
   - Special handling for `InterruptedException`: checks if `Thread.currentThread().interrupt()` is invoked.
3. If neither throw nor log is found, a `SwallowedExceptionFinding` is emitted.

---

### False Positive (FP) Handling & Mitigations
1. **Standard `ignored` / `expected` Parameter Conventions:**
   - In JUnit tests or fallback parsing (e.g. attempting date format A then falling back to format B), developers intentionally write `catch (ParseException ignored) { ... }`. AG008 respects this standard convention and does **not** flag it.
2. **Restoring Thread Interruption:**
   - For `InterruptedException`, calling `Thread.currentThread().interrupt()` is the gold-standard compliant pattern for virtual and carrier threads. AG008 detects `interrupt()` calls and marks the catch block as compliant.
3. **Structured Logging Calls:**
   - Matches any SLF4J, Log4j, or custom logger invocation (`log.warn(...)`, `logger.error(...)`).
4. **Controlled Fallback & Functional Defaults:**
   - Catch blocks handling expected parsing/conversion failures (e.g. `NumberFormatException`, `DateTimeParseException`, `NoSuchElementException`) that return safe functional fallbacks (`return false`, `return Optional.empty()`, `return defaultValue`) are recognized as intentional error handling and not flagged.

---

### False Negative (FN) Handling & Defense-in-Depth
1. **Dummy Statements in Catch:**
   - Catch blocks that contain `int x = 0;` or a comment without logging or throwing are accurately flagged.
2. **Nested Try-Catch Blocks:**
   - AST node traversal recursively inspects nested catch clauses in complex error recovery pipelines.

---

## Code Examples

### Non-Compliant (Violating Anti-Pattern)

```java
// VIOLATION: Empty catch block hides database failure
public void saveUserData(User user) {
    try {
        userRepository.save(user);
    } catch (DataAccessException e) {
        // Exception swallowed! No log, no rethrow.
    }
}

// VIOLATION: Catch block prints text without logging the exception
try {
    processPayment();
} catch (Exception e) {
    System.out.println("Payment failed"); // Loss of stack trace and logger context!
}
```

### Compliant (Refactored Best Practice)

```java
// COMPLIANT: Properly logged with SLF4J and rethrown
public void saveUserData(User user) {
    try {
        userRepository.save(user);
    } catch (DataAccessException e) {
        log.error("Failed to persist user {}: {}", user.getId(), e.getMessage(), e);
        throw new DomainPersistenceException("Could not save user", e);
    }
}

// COMPLIANT: Restoring interrupted status
public void waitWorker() {
    try {
        Thread.sleep(1000);
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt(); // Proper thread management
    }
}
```

### Legitimate Edge Cases Handled Without Alerting

```java
// COMPLIANT: Explicitly documented intentional ignore
try {
    return Integer.parseInt(input);
} catch (NumberFormatException ignored) {
    return 0; // Compliant: parameter named 'ignored' signals deliberate fallback
}
```

---

## Configuration & Tuning

In `archguard.yml`, you can customize rule severity, exclude specific classes, or adjust thresholds:

```yaml
rules:
  AG008:
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
@SuppressArchitecture("AG008")
public void myMethod() {
    // ...
}

// Or using standard Java @SuppressWarnings
@SuppressWarnings("archguard:AG008")
public void myMethod() {
    // ...
}
```
