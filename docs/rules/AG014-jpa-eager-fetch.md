# AG014: JPA Eager Fetch Detection

| Attribute | Details |
| :--- | :--- |
| **Rule ID** | `AG014` |
| **Name** | JPA Eager Fetch Detection |
| **Default Severity** | `WARNING` |
| **Category** | Database Performance |
| **Analysis Engine** | AST Source Analysis (JavaParser) |
| **Core Classes** | `AG014EagerFetchRule, PerformanceInspector` |

---

## Executive Summary & Architectural Rationale

Eager fetching in JPA/Hibernate is a primary driver of database performance degradation:
1. **Cartesian Product Explosions:** Eagerly fetching multiple collection relationships causes Hibernate to execute multi-table SQL joins, producing combinatorial duplicate rows in memory.
2. **Severe N+1 Query Overheads:** When querying a list of parent entities, eager associations trigger individual sub-queries for every single row, multiplying database roundtrips.
3. **Heap Memory Starvation:** Loading entire entity relationship graphs into the JVM first-level cache leads to high GC pause times and OutOfMemoryErrors.

---

## How It Is Implemented

AST analysis inspects fields in `@Entity` and `@Document` classes:
1. Checks relationship annotations: `@OneToMany`, `@ManyToMany`, `@ManyToOne`, `@OneToOne`.
2. Flags explicit `FetchType.EAGER` configurations.
3. Flags `@ManyToOne` and `@OneToOne` where default is eager unless explicitly specified as `FetchType.LAZY`.

---

### False Positive (FP) Handling & Mitigations
1. **Explicit `FetchType.LAZY`:**
   - Any relationship explicitly configured with `(fetch = FetchType.LAZY)` is compliant and ignored.
2. **Targeted Fetching Alternatives:**
   - Developers are encouraged to use targeted JPQL `JOIN FETCH` or Spring Data JPA `@EntityGraph` per query instead of global eager loading.

---

### False Negative (FN) Handling & Defense-in-Depth
1. **Default Eager Mappings:**
   - Recognizes that JPA defaults `@ManyToOne` to `FetchType.EAGER` unless overridden, warning developers of hidden eager joins.

---

## Code Examples

### Non-Compliant (Violating Anti-Pattern)

```java
// VIOLATION: Eager fetching on collection relationship
@Entity
public class OrderEntity {

    @OneToMany(mappedBy = "order", fetch = FetchType.EAGER) // Cartesian product risk!
    private List<OrderItemEntity> items = new ArrayList<>();
}
```

### Compliant (Refactored Best Practice)

```java
// COMPLIANT: Lazy fetching with query-level JOIN FETCH
@Entity
public class OrderEntity {

    @OneToMany(mappedBy = "order", fetch = FetchType.LAZY) // Clean lazy loading
    private List<OrderItemEntity> items = new ArrayList<>();
}

// In Repository: Fetch eagerly only when explicitly needed
public interface OrderRepository extends JpaRepository<OrderEntity, Long> {
    @EntityGraph(attributePaths = {"items"})
    Optional<OrderEntity> findWithItemsById(Long id);
}
```

### Legitimate Edge Cases Handled Without Alerting

```java
// COMPLIANT: Lazy fetch explicitly set on ManyToOne
@Entity
public class OrderItemEntity {
    @ManyToOne(fetch = FetchType.LAZY) // Overridden default: compliant
    private OrderEntity order;
}
```

---

## Configuration & Tuning

In `archguard.yml`, you can customize rule severity, exclude specific classes, or adjust thresholds:

```yaml
rules:
  AG014:
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
@SuppressArchitecture("AG014")
public void myMethod() {
    // ...
}

// Or using standard Java @SuppressWarnings
@SuppressWarnings("archguard:AG014")
public void myMethod() {
    // ...
}
```
