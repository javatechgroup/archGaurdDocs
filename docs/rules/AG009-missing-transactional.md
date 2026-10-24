# AG009: Missing @Transactional on Multi-Repository Writes

| Attribute | Details |
| :--- | :--- |
| **Rule ID** | `AG009` |
| **Name** | Missing @Transactional on Multi-Repository Writes |
| **Default Severity** | `WARNING` |
| **Category** | Data Integrity & Transactions |
| **Analysis Engine** | AST Source Analysis (JavaParser) |
| **Core Classes** | `AG009MissingTransactionalRule, CodeMetricsInspector` |

---

## Executive Summary & Architectural Rationale

Enterprise database integrity depends on atomic transactions:
1. **Partial Write Corruption:** When a service method writes to `orderRepository.save(...)` and subsequently to `inventoryRepository.deduct(...)` without `@Transactional`, if an unexpected exception occurs during the inventory call, the order commit remains persisted in the database!
2. **Phantom Commits & Missing Rollbacks:** Without a transaction coordinator, auto-commit mode persists each statement independently, leaving database state permanently inconsistent.
3. **Audit Inconsistency:** Financial records, inventory counts, and audit logs desynchronize.

---

## How It Is Implemented

AST inspection evaluates public methods in `@Service` beans:
1. **Service Bean Identification:** Inspects classes with `@Service` or `*Service` / `*ServiceImpl`.
2. **Class-Level Transaction Check:** If class has `@Transactional`, all public methods are transactional by default (exempted).
3. **Mutating Call Collector:**
   - Analyzes method calls where scope ends with `Repository`, `Repo`, or `Dao`.
   - Filters out read-only calls using `READ_ONLY_REPO_PREFIXES`: `find`, `get`, `read`, `query`, `search`, `count`, `exists`, `select`, `stream`, `load`, `list`, `fetch`, `check`.
   - Collects remaining mutating calls (`save`, `delete`, `update`, `insert`, `flush`).
4. **Threshold:** If $\ge 2$ distinct mutating repository operations occur without `@Transactional`, records `MissingTransactionalFinding`.

---

### False Positive (FP) Handling & Mitigations
1. **Class-Level `@Transactional`:**
   - If the class itself has `@Transactional` or `@Transactional(readOnly = false)`, all contained methods inherit the transaction boundary. AG009 checks class annotations first.
2. **Read-Only Lookups:**
   - Methods performing multiple reads (e.g. `userRepo.findById(id)` and `orderRepo.findByUserId(id)`) do **not** mutate data and are ignored.
3. **Single Repository Operations:**
   - A method calling a single `orderRepo.save(order)` is excluded because atomic single-record writes do not present multi-table atomicity hazards.

---

### False Negative (FN) Handling & Defense-in-Depth
1. **Repository Scope Aliases:**
   - Recognizes standard repository naming suffixes (`*Repository`, `*Repo`, `*Dao`).
2. **Different Methods on Same or Different Repositories:**
   - Tracks unique method invocation signatures (`scope.methodName`), catching multiple mutating writes across both single and multiple repositories.

---

## Code Examples

### Non-Compliant (Violating Anti-Pattern)

```java
// VIOLATION: Two mutating repository calls without @Transactional boundary
@Service
public class OrderService {

    private final OrderRepository orderRepository;
    private final AuditRepository auditRepository;

    public void placeOrder(Order order) { // Missing @Transactional!
        orderRepository.save(order);      // First mutation persists
        auditRepository.recordLog(order); // If this throws, order is NOT rolled back!
    }
}
```

### Compliant (Refactored Best Practice)

```java
// COMPLIANT: Explicit @Transactional ensures atomic all-or-nothing rollback
@Service
public class OrderService {

    private final OrderRepository orderRepository;
    private final AuditRepository auditRepository;

    @Transactional
    public void placeOrder(Order order) {
        orderRepository.save(order);
        auditRepository.recordLog(order);
    }
}
```

### Legitimate Edge Cases Handled Without Alerting

```java
// COMPLIANT: Multiple repository calls that are strictly read-only queries
@Service
public class OrderQueryService {
    public OrderSummary getSummary(Long id) {
        var user = userRepository.findById(id).orElseThrow();   // Read-only
        var orders = orderRepository.findByUserId(id);          // Read-only
        return new OrderSummary(user, orders);
    }
}
```

---

## Configuration & Tuning

In `archguard.yml`, you can customize rule severity, exclude specific classes, or adjust thresholds:

```yaml
rules:
  AG009:
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
@SuppressArchitecture("AG009")
public void myMethod() {
    // ...
}

// Or using standard Java @SuppressWarnings
@SuppressWarnings("archguard:AG009")
public void myMethod() {
    // ...
}
```
