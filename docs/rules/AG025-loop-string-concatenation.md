# AG025: Loop String Concatenation

| Attribute | Details |
| :--- | :--- |
| **Rule ID** | `AG025` |
| **Name** | Loop String Concatenation |
| **Default Severity** | `WARNING` |
| **Category** | Performance & Memory |
| **Analysis Engine** | AST Source Analysis (JavaParser) |
| **Core Classes** | `AG025LoopStringConcatenationRule, PerformanceInspector` |

---

## Executive Summary & Architectural Rationale

Repeated string concatenation inside loops causes severe JVM memory churn:
1. **$O(n^2)$ Memory Copying:** Java strings are immutable. Every `str += item` inside a loop creates a new `StringBuilder`, appends the strings, calls `toString()`, and copies the entire accumulated character array into a newly allocated `String` object.
2. **Excessive Garbage Collection (GC) Pressure:** For a loop of 1,000 iterations, $O(n^2)$ intermediate string objects are allocated and discarded on the Young Generation heap, triggering frequent stop-the-world Young GC pauses.
3. **Throughput Degradation:** Latency increases non-linearly as loop size increases.

---

## How It Is Implemented

AST analysis inspects all loop nodes (`ForStmt`, `ForEachStmt`, `WhileStmt`, `DoStmt`):
1. **Assignment Expression Inspection:** Finds `AssignExpr` where operator is `PLUS` (`+=`).
2. **Type Disambiguation (`isStringConcatenation`):**
   - Checks target variable declared type: if numeric (`int`, `long`, `double`, `BigDecimal`, etc.), skips immediately (exempted).
   - If target type is `String` or value is a string literal / `.toString()` call, flags as string concatenation.
   - If target variable name indicates text accumulator (`str`, `msg`, `text`, `csv`, `report`) and value is not a numeric literal, flags as string concatenation.
3. Records `LoopStringConcatenationFinding`.

---

### False Positive (FP) Handling & Mitigations
1. **Numeric Compound Additions:**
   - Expressions like `total += item.getPrice()` or `sum += 1` use the `+` operator for arithmetic addition. `isStringConcatenation` checks the variable type against `NUMERIC_TYPES` and exempts all numeric accumulators.
2. **Static Single-Line String Concatenation Outside Loops:**
   - String concatenation outside of loops (e.g. `String msg = "Hello " + name;`) is automatically optimized by `javac` into `makeConcatWithConstants` (Java 9+ invokedynamic) and is not flagged.

---

### False Negative (FN) Handling & Defense-in-Depth
1. **All Loop Variants:**
   - Evaluates `for`, enhanced `for-each`, `while`, and `do-while` loops.
2. **Inferred String Variables:**
   - Detects concatenation even when concatenating variables whose types are resolved from method scopes.

---

## Code Examples

### Non-Compliant (Violating Anti-Pattern)

```java
// VIOLATION: String concatenation inside loop generates O(n^2) garbage
public String buildCsv(List<Order> orders) {
    String csv = "";
    for (Order order : orders) {
        csv += order.getId() + "," + order.getTotal() + "
"; // Heap memory churn!
    }
    return csv;
}
```

### Compliant (Refactored Best Practice)

```java
// COMPLIANT: Using StringBuilder with pre-sized or dynamic buffer
public String buildCsv(List<Order> orders) {
    StringBuilder sb = new StringBuilder();
    for (Order order : orders) {
        sb.append(order.getId()).append(",").append(order.getTotal()).append("
");
    }
    return sb.toString();
}

// ALSO COMPLIANT: Java Stream Collectors
public String buildCsvStream(List<Order> orders) {
    return orders.stream()
            .map(o -> o.getId() + "," + o.getTotal())
            .collect(Collectors.joining("
"));
}
```

### Legitimate Edge Cases Handled Without Alerting

```java
// COMPLIANT: Numeric accumulation in loop
int totalSum = 0;
for (int val : numbers) {
    totalSum += val; // Arithmetic addition: recognized and exempted
}
```

---

## Configuration & Tuning

In `archguard.yml`, you can customize rule severity, exclude specific classes, or adjust thresholds:

```yaml
rules:
  AG025:
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
@SuppressArchitecture("AG025")
public void myMethod() {
    // ...
}

// Or using standard Java @SuppressWarnings
@SuppressWarnings("archguard:AG025")
public void myMethod() {
    // ...
}
```
