# AG018: High Cyclomatic Complexity

| Attribute | Details |
| :--- | :--- |
| **Rule ID** | `AG018` |
| **Name** | High Cyclomatic Complexity |
| **Default Severity** | `WARNING` |
| **Category** | Maintainability & Code Quality |
| **Analysis Engine** | AST Source Analysis (JavaParser) |
| **Core Classes** | `AG018MethodComplexityRule, MethodComplexityInspector` |

---

## Executive Summary & Architectural Rationale

High cyclomatic complexity directly correlates with bug density and maintenance cost:
1. **Exponential Execution Paths:** A method with complexity $> 15$ has hundreds of execution path combinations, making complete unit test branch coverage nearly impossible.
2. **Cognitive Overload:** Developers cannot mentally trace highly branched logic, leading to regression bugs during maintenance.
3. **Testing Drag:** Requires dozens of mock permutations to verify all error handling branches.

---

## How It Is Implemented

Calculates McCabe cyclomatic complexity for each `MethodDeclaration` using JavaParser:
1. Base complexity = 1.
2. Increments for control-flow branching statements:
   - `IfStmt` (+1)
   - `ForStmt`, `ForEachStmt`, `WhileStmt`, `DoStmt` (+1 each)
   - `CatchClause` (+1 each)
   - `ConditionalExpr` (ternary `? :`) (+1 each)
   - `SwitchEntry` labels (+1 per case label)
   - Logical operators: `BinaryExpr` with `&&` or `||` (+1 each)
3. If total complexity $> 15$, records `MethodComplexityFinding`.

---

### False Positive (FP) Handling & Mitigations
1. **Label-Level Switch Accounting:**
   - Accurately counts individual case labels rather than compounding entire switch blocks.
2. **Configurable Threshold:**
   - Threshold defaults to 15, matching industry standards (SonarQube, PMD). Can be configured in `archguard.yml`.

---

### False Negative (FN) Handling & Defense-in-Depth
1. **Compound Boolean Expressions:**
   - Naive analyzers ignore `&&` and `||`. AG018 inspects `BinaryExpr` to account for short-circuit evaluation branches accurately.

---

## Code Examples

### Non-Compliant (Violating Anti-Pattern)

```java
// VIOLATION: Method with cyclomatic complexity > 15
public boolean validateOrder(Order o) {
    if (o == null) return false;
    if (o.getId() == null || o.getId() <= 0) return false;
    if (o.getItems() == null || o.getItems().isEmpty()) return false;
    for (Item item : o.getItems()) {
        if (item.getPrice() < 0 || item.getQuantity() <= 0) return false;
        if (item.isFragile() && !o.isSpecialHandling()) return false;
    }
    // Deep conditional branching continues...
    return true;
}
```

### Compliant (Refactored Best Practice)

```java
// COMPLIANT: Refactored with polymorphic validators or Specification pattern
public boolean validateOrder(Order order) {
    return orderValidators.stream().allMatch(v -> v.isValid(order));
}
```

### Legitimate Edge Cases Handled Without Alerting

```java
// COMPLIANT: Simple method with complexity = 3
public String getStatus(int code) {
    if (code == 200) return "OK";
    return code == 404 ? "NOT_FOUND" : "ERROR";
}
```

---

## Configuration & Tuning

In `archguard.yml`, you can customize rule severity, exclude specific classes, or adjust thresholds:

```yaml
rules:
  AG018:
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
@SuppressArchitecture("AG018")
public void myMethod() {
    // ...
}

// Or using standard Java @SuppressWarnings
@SuppressWarnings("archguard:AG018")
public void myMethod() {
    // ...
}
```
