# AG023: NullPointer Traps & Defensive Coding

| Attribute | Details |
| :--- | :--- |
| **Rule ID** | `AG023` |
| **Name** | NullPointer Traps & Defensive Coding |
| **Default Severity** | `WARNING` |
| **Category** | Defensive Programming |
| **Analysis Engine** | AST Source Analysis (JavaParser) |
| **Core Classes** | `AG023NullSafetyRule, ResourceAndSafetyInspector` |

---

## Executive Summary & Architectural Rationale

`NullPointerException` and `NoSuchElementException` are the leading causes of production downtime in Java:
1. **Raw `Optional.get()`:** Defeats the entire purpose of `Optional`. If the optional is empty, calling `.get()` immediately throws `NoSuchElementException`.
2. **Returning `null` Collections:** Returning `null` instead of an empty list or empty map forces every caller to add defensive null checks. Callers that forget will crash with `NullPointerException` when iterating with `for (Item item : list)`.
3. **Inverted String Comparison:** Writing `variable.equals("LITERAL")` crashes with NPE if `variable` is null. Writing `"LITERAL".equals(variable)` is guaranteed null-safe.

---

## How It Is Implemented

AST inspection in `ResourceAndSafetyInspector`:
1. **Returning Null Collections / Optional:**
   - Inspects methods with return types `List`, `Set`, `Map`, `Collection`, `Optional`, `Iterable`.
   - Inspects `ReturnStmt` nodes: flags explicit `return null;`.
2. **Raw `Optional.get()` Inspection:**
   - Flags calls to `.get()` with zero arguments on optional-like scopes.
   - Verifies if call is enclosed in an `IfStmt` checking `.isPresent()` or `!.isEmpty()`. If guarded, exempts.
3. **Inverted String Literal Comparison:**
   - Checks `equals()` or `equalsIgnoreCase()` where argument is a `StringLiteralExpr` and scope is a variable.
   - Checks if preceded by null check (`var != null && var.equals(...)`). If guarded, exempts.

---

### False Positive (FP) Handling & Mitigations
1. **Guarded `Optional.isPresent()` & Early-Return Guard Clauses:**
   - Calls enclosed in `if (optional.isPresent()) { return optional.get(); }` or protected by early returns (e.g. `if (optional.isEmpty()) return; return optional.get();`) are recognized as safe and are not flagged.
2. **Non-Optional `.get()` Invocations:**
   - Distinguishes non-`Optional` types (`CompletableFuture`, `AtomicReference`, `Supplier`, `ThreadLocal`) from `Optional.get()`, preventing false alerts on concurrency and functional primitives.
3. **Guarded Null Comparisons:**
   - If written as `if (status != null && status.equals("ACTIVE"))`, the binary expression short-circuits safely and is not flagged.
4. **Yoda Comparisons:**
   - `"ACTIVE".equals(status)` has literal as scope and is fully compliant.

---

### False Negative (FN) Handling & Defense-in-Depth
1. **Both Optional and Collections Checked:**
   - Enforces empty collections for `List`, `Set`, `Map`, `Collection`, `Iterable`, and `Optional.empty()`.
2. **Matches Both `equals()` and `equalsIgnoreCase()`:**
   - Catches both comparison variants.

---

## Code Examples

### Non-Compliant (Violating Anti-Pattern)

```java
// VIOLATION: Returning null collection, raw Optional.get(), and inverted equals
public List<User> getUsers() {
    return null; // Forces callers to crash or null-check!
}

public User getUser(Long id) {
    Optional<User> userOpt = userRepository.findById(id);
    return userOpt.get(); // Throws NoSuchElementException if missing!
}

public boolean isActive(String status) {
    return status.equals("ACTIVE"); // Throws NPE if status is null!
}
```

### Compliant (Refactored Best Practice)

```java
// COMPLIANT: Empty collections, orElseThrow, and null-safe Yoda comparison
public List<User> getUsers() {
    return Collections.emptyList(); // Safe for callers to iterate
}

public User getUser(Long id) {
    return userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id)); // Explicit error
}

public boolean isActive(String status) {
    return "ACTIVE".equals(status); // Null-safe Yoda equality
}
```

### Legitimate Edge Cases Handled Without Alerting

```java
// COMPLIANT: Optional.get() guarded by isPresent()
if (optionalUser.isPresent()) {
    log.info("Found user: {}", optionalUser.get().getName()); // Guarded: permitted
}
```

---

## Configuration & Tuning

In `archguard.yml`, you can customize rule severity, exclude specific classes, or adjust thresholds:

```yaml
rules:
  AG023:
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
@SuppressArchitecture("AG023")
public void myMethod() {
    // ...
}

// Or using standard Java @SuppressWarnings
@SuppressWarnings("archguard:AG023")
public void myMethod() {
    // ...
}
```
