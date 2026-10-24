# AG019: Algorithmic Performance Risk in Loops

| Attribute | Details |
| :--- | :--- |
| **Rule ID** | `AG019` |
| **Name** | Algorithmic Performance Risk in Loops |
| **Default Severity** | `WARNING` |
| **Category** | Runtime Performance |
| **Analysis Engine** | AST Source Analysis (JavaParser) |
| **Core Classes** | `AG019AlgorithmicPerformanceRule, PerformanceInspector` |

---

## Executive Summary & Architectural Rationale

Algorithmic traps inside loops are frequent causes of production performance outages:
1. **$O(n^2)$ CPU Bottlenecks:** Calling `list.contains(x)` or `list.indexOf(x)` inside a loop iterating over $n$ items requires $O(n \times m)$ comparisons. When collections grow from 100 to 10,000 elements, execution time jumps from milliseconds to minutes.
2. **N+1 Database Query Storms:** Executing `repository.findById(id)` or `repository.save(entity)` inside a loop sends hundreds of individual network roundtrips to PostgreSQL/MySQL, saturating database CPU and connection pools.

---

## How It Is Implemented

AST inspection traverses all loop statements (`ForStmt`, `ForEachStmt`, `WhileStmt`, `DoStmt`):
1. **Variable Type Map:** Collects field, parameter, and local variable declared types.
2. **Collection Lookup Inspection:**
   - Identifies calls to `contains()` or `indexOf()`.
   - Filters out safe lookups via `PerformanceInspector.isCollectionLookup()`:
     - Excludes `Set` / `HashSet` ($O(1)$ constant-time lookup).
     - Excludes `String.contains()` and `String.indexOf('c')`.
     - Excludes String transformations (`.toLowerCase().contains()`).
   - If confirmed linear collection lookup, records finding.
3. **Database Query in Loop Inspection:**
   - Identifies repository calls on scopes ending in `Repository`, `Repo`, `Dao`.
   - Flags `find*`, `get*`, `save*`, `delete*` called inside loop bodies.

---

### False Positive (FP) Handling & Mitigations
1. **$O(1)$ `Set.contains()` Lookups:**
   - Set lookups (`set.contains(id)`) execute in $O(1)$ constant time and are the recommended compliant pattern. `isCollectionLookup` explicitly checks if scope is a Set and exempts it.
2. **String Operations:**
   - Methods like `"hello".contains("ell")` or `name.indexOf('/')` operate on strings, not collections. The inspector verifies argument types (char literals, string literals on non-collections) and variable types to exclude string operations.
3. **String Getters:**
   - Calls like `user.getName().contains("Admin")` are recognized as string operations and ignored.
4. **Bounded Retry and Collision-Resolution Loops:**
   - Loops designed for retry mechanisms or collision-checking with a bounded iteration counter (e.g., `for (int attempt = 0; attempt <= MAX_RETRIES; attempt++)` or `while (attempt < 5)`) where a single query checks existence (e.g., `repo.findByCode(code)`) are recognized via loop variable naming (`attempt`, `retry`, `tries`) or small bounded conditions ($\le 10$). These are legitimate retry/collision loops, not N+1 batch iteration queries over collections.

---

### False Negative (FN) Handling & Defense-in-Depth
1. **Implicit Collection Names:**
   - If variable type is omitted or generic, heuristic checks target names (`ids.contains(...)`, `items.indexOf(...)`) to ensure evasion is prevented.
2. **Both Query and Write Repositories:**
   - Catches both N+1 query loops (`repo.findById`) and N+1 write loops (`repo.save`).

---

## Code Examples

### Non-Compliant (Violating Anti-Pattern)

```java
// VIOLATION: O(n^2) List lookup and N+1 database queries in loop
public void processOrders(List<Order> orders, List<String> blacklistedIds) {
    for (Order order : orders) {
        if (blacklistedIds.contains(order.getUserId())) { // O(n^2) linear scan on List!
            continue;
        }
        orderRepository.save(order); // N+1 database roundtrip inside loop!
    }
}
```

### Compliant (Refactored Best Practice)

```java
// COMPLIANT: O(1) Set lookup and batch database query outside loop
public void processOrders(List<Order> orders, List<String> blacklistedIds) {
    Set<String> blacklistedSet = new HashSet<>(blacklistedIds); // O(1) lookup structure

    List<Order> validOrders = orders.stream()
            .filter(o -> !blacklistedSet.contains(o.getUserId()))
            .toList();

    orderRepository.saveAll(validOrders); // Single batch write
}
```

### Legitimate Edge Cases Handled Without Alerting

```java
// COMPLIANT: String.contains() inside loop
for (String line : logLines) {
    if (line.contains("ERROR")) { // String inspection: NOT a collection lookup
        errorCount++;
    }
}

// COMPLIANT: Bounded retry / collision-resolution loop
int attempt = 0;
while (attempt < 5) {
    String uniqueCode = generateCode();
    if (!orderRepository.existsByCode(uniqueCode)) { // Permitted: bounded retry check
        return uniqueCode;
    }
    attempt++;
}
```

---

## Configuration & Tuning

In `archguard.yml`, you can customize rule severity, exclude specific classes, or adjust thresholds:

```yaml
rules:
  AG019:
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
@SuppressArchitecture("AG019")
public void myMethod() {
    // ...
}

// Or using standard Java @SuppressWarnings
@SuppressWarnings("archguard:AG019")
public void myMethod() {
    // ...
}
```
