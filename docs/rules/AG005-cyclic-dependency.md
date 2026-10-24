# AG005: Class-Level Cyclic Dependency

| Attribute | Details |
| :--- | :--- |
| **Rule ID** | `AG005` |
| **Name** | Class-Level Cyclic Dependency |
| **Default Severity** | `ERROR` |
| **Category** | Coupling & Maintainability |
| **Analysis Engine** | Graph Theory (Tarjan's SCC + DFS Cycle Tracing) on Bytecode Graph |
| **Core Classes** | `AG005CycleDetectionRule, CycleDetector, DependencyGraph` |

---

## Executive Summary & Architectural Rationale

Circular dependencies between classes represent severe architectural degradation:
1. **Initialization Deadlocks & Order Fragility:** Spring must use circular reference resolvers (such as `@Lazy` or reflection proxy fields) to initialize beans in cycles. In Spring Boot 2.6+, circular bean references are disabled by default (`spring.main.allow-circular-references=false`) and cause application startup crashes.
2. **Impossible Unit Testing:** Neither class can be tested or reasoned about in isolation without instantiating or mocking the other.
3. **Monolithic Rigidity:** Cycles destroy the Directed Acyclic Graph (DAG) property of clean software architecture, preventing modularization and microservice decomposition.

---

## How It Is Implemented

Operates on the complete directed `DependencyGraph` extracted from bytecode:
1. **Tarjan's Strongly Connected Components (SCC) Algorithm:**
   - Identifies maximal subgraphs where every vertex is reachable from every other vertex in $O(V + E)$ time.
   - Any SCC containing $\ge 2$ nodes (or 1 node with a self-loop) contains at least one cycle.
2. **Depth-First Search (DFS) Cycle Path Tracing:**
   - Traverses each multi-node SCC to trace the exact sequence of classes in each elementary cycle.
   - Path search is bounded at depth 15 to prevent exponential path explosions on dense legacy graphs.
3. **Canonical Cycle Normalization:**
   - Cycles are rotated so they always start at the lexicographically smallest class name (e.g. `B -> C -> A -> B` is normalized to `A -> B -> C -> A`).
   - Deduplicates identical cycles found from different starting points during DFS traversal.

---

### False Positive (FP) Handling & Mitigations
1. **Duplicate Cycle Representations:**
   - In graph theory, $A \to B \to A$ and $B \to A \to B$ are the same cycle. Without canonical normalization, static analysis tools report 2 separate violations for the same 2-node cycle. AG005 normalizes cycles lexicographically, reporting exactly **one** canonical violation per physical cycle.
2. **Third-Party & Framework Dependencies:**
   - The dependency graph only includes nodes within the analyzed project classes. Calls to Java runtime (`java.util.*`) or Spring framework libraries never generate cycles.
3. **Self-Referential Types vs Cyclic Coupling:**
   - A class having a self-reference (e.g. a tree node `public TreeNode parent;`) is handled separately as a self-edge, preventing misleading multi-class cycle alerts.

---

### False Negative (FN) Handling & Defense-in-Depth
1. **Interface & Indirect Inversion Cycles:**
   - If `OrderService` depends on `PaymentService`, and `PaymentServiceImpl` depends on `OrderService`, bytecode graph captures all 7 dependency types (interfaces, implementations, constructor parameters, method calls, field types), detecting indirect cycles that source code AST tools miss.
2. **Lombok Constructor Injected Cycles:**
   - Cycles created via Lombok's `@RequiredArgsConstructor` appear only as constructor parameters in compiled bytecode. ArchUnit's `JavaConstructor` inspection reliably extracts them.
3. **Transitive Cycle Chains:**
   - Catches long cycle chains ($A \to B \to C \to D \to E \to A$) up to 15 nodes deep via Tarjan's SCC decomposition.

---

## Code Examples

### Non-Compliant (Violating Anti-Pattern)

```java
// VIOLATION: Circular reference between OrderService and PaymentService
@Service
public class OrderService {
    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}

@Service
public class PaymentService {
    private final OrderService orderService; // CYCLE: OrderService -> PaymentService -> OrderService

    public PaymentService(OrderService orderService) {
        this.orderService = orderService;
    }
}
```

### Compliant (Refactored Best Practice)

```java
// COMPLIANT: Decoupled via Dependency Inversion or Event-Driven Messaging
@Service
public class OrderService {
    private final ApplicationEventPublisher eventPublisher;

    public OrderService(ApplicationEventPublisher eventPublisher) {
        this.eventPublisher = eventPublisher;
    }

    public void completeOrder(Order order) {
        eventPublisher.publishEvent(new OrderCompletedEvent(order.getId()));
    }
}

@Component
public class PaymentEventListener {
    @EventListener
    public void onOrderCompleted(OrderCompletedEvent event) {
        // Process payment without direct cyclic coupling
    }
}
```

### Legitimate Edge Cases Handled Without Alerting

```java
// COMPLIANT: Self-referential tree structure (not a class-to-class cycle)
public class Department {
    private Long id;
    private Department parent; // Self-reference in domain model: permitted
    private List<Department> children = new ArrayList<>();
}
```

---

## Configuration & Tuning

In `archguard.yml`, you can customize rule severity, exclude specific classes, or adjust thresholds:

```yaml
rules:
  AG005:
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
@SuppressArchitecture("AG005")
public void myMethod() {
    // ...
}

// Or using standard Java @SuppressWarnings
@SuppressWarnings("archguard:AG005")
public void myMethod() {
    // ...
}
```
