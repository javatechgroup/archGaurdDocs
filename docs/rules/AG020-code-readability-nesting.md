# AG020: Code Readability & Nesting

| Attribute | Details |
| :--- | :--- |
| **Rule ID** | `AG020` |
| **Name** | Code Readability & Nesting |
| **Default Severity** | `WARNING` |
| **Category** | Clean Code |
| **Analysis Engine** | AST Source Analysis (JavaParser) |
| **Core Classes** | `AG020CodeReadabilityRule, MethodComplexityInspector` |

---

## Executive Summary & Architectural Rationale

Code readability smells represent severe maintainability debt:
1. **Monster Methods (> 60 lines):** Oversized methods violate the Single Responsibility Principle, interweaving data extraction, validation, orchestration, and persistence into unreadable monolithic blocks.
2. **Arrow Anti-Pattern (Nesting >= 4 levels):** Deeply nested `if/for/try` blocks drift code off the right side of the screen, creating confusion around which condition governs which block.
3. **Long Parameter Lists (>= 5 params):** Long parameter lists are error-prone, especially when passing multiple arguments of the same type (`String first, String last, String email`).
4. **Bloated Classes (> 25 methods or > 450 lines):** Classes that accumulate too many responsibilities become monolithic maintenance bottlenecks.

---

## How It Is Implemented

AST analysis inspects class structure and method declarations:
1. **Executable Line Count (`countExecutableLines`):**
   - Strips blank lines, single-line comments (`//`), block comments (`/* ... */`), Javadoc, and standalone braces (`{`, `}`).
   - Measures pure executable code lines. Flags methods $> 60$ lines and classes $> 450$ lines.
2. **Method Count:** Flags non-data classes with $> 25$ methods.
3. **Parameter Count:** Flags methods with $\ge 5$ parameters.
4. **Nesting Depth Calculation (`calculateMaxNestingDepth`):**
   - Measures indentation levels for `if`, `for`, `while`, `do`, `try`, `switch`.
   - Special handling for `else if`: treats `else if` as flat branching, not nested depth.

---

### False Positive (FP) Handling & Mitigations
1. **Flat `else if` Ladders:**
   - An `else if` block technically nests inside the `else` statement of an AST node. AG020 explicitly checks `parentIf.getElseStmt() == node` and does **not** increment nesting depth for `else if` ladders, preventing false positives on multi-condition dispatchers.
2. **Well-Commented Code & Javadoc:**
   - Pure lines of code count ignores Javadoc, license headers, and whitespace, ensuring documentation does not penalize line count metrics.
3. **Constructors & Factory Methods:**
   - Can use Records or DTOs to cleanly bundle parameters.
4. **Spring Data Repository Query Methods:**
   - Spring Data JPA/Mongo repository interfaces (`@Repository` or `*Repository` / `*Dao`) regularly declare multi-criteria query methods (e.g., `findByCityAndCategoryAndStatusAndStartDateAndEndDate(...)`). Because query method parameters are dictated by Spring Data query-derivation conventions and interfaces have no executable implementation bodies, parameter count checking ($\ge 5$) is exempted for repository interfaces.
5. **Data Models, DTOs, and Security Models (`UserDetails`):**
   - Classes where $\ge 70\%$ of methods are standard getters, setters, or contract methods (e.g. `UserDetails`, `Principal`, DTOs, POJOs, entity models) are data carriers, not orchestrators. They are exempted from the "Excessive method count in class (> 25 methods)" smell, which is reserved for bloated monolithic services.

---

### False Negative (FN) Handling & Defense-in-Depth
1. **Combined Nesting Constructs:**
   - Accurately computes cumulative depth across mixed constructs: `try { for { if { while { ... } } } }` is correctly calculated as depth 4.

---

## Code Examples

### Non-Compliant (Violating Anti-Pattern)

```java
// VIOLATION: Deep nesting (Arrow Anti-Pattern, depth >= 4)
public void processRecord(Record r) {
    if (r != null) {
        if (r.isValid()) {
            for (Item item : r.getItems()) {
                if (item.isAvailable()) { // Depth 4!
                    reserveItem(item);
                }
            }
        }
    }
}
```

### Compliant (Refactored Best Practice)

```java
// COMPLIANT: Flattened using Guard Clauses (Return Early)
public void processRecord(Record r) {
    if (r == null || !r.isValid()) {
        return; // Guard clause
    }
    for (Item item : r.getItems()) {
        if (!item.isAvailable()) {
            continue; // Guard clause
        }
        reserveItem(item); // Flat execution flow
    }
}
```

### Legitimate Edge Cases Handled Without Alerting

```java
// COMPLIANT: Flat else-if chain is not penalized as deep nesting
if (status == 1) {
    handleInit();
} else if (status == 2) {
    handleActive();
} else if (status == 3) {
    handleClosed(); // Evaluated at depth 1: permitted
}

// COMPLIANT: Spring Data Repository query method with multi-column criteria
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    List<Order> findByCustomerIdAndStatusAndOrderDateBetweenAndRegion(
            Long customerId, OrderStatus status, LocalDate start, LocalDate end, String region);
}

// COMPLIANT: Security UserDetails / DTO with many getter/setter pairs
public class UserPrincipal implements UserDetails {
    private String username;
    private String password;
    // 14+ fields with getters and setters: permitted as data carrier
}
```

---

## Configuration & Tuning

In `archguard.yml`, you can customize rule severity, exclude specific classes, or adjust thresholds:

```yaml
rules:
  AG020:
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
@SuppressArchitecture("AG020")
public void myMethod() {
    // ...
}

// Or using standard Java @SuppressWarnings
@SuppressWarnings("archguard:AG020")
public void myMethod() {
    // ...
}
```
