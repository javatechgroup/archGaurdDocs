# AG003: Service to Controller Inverted Dependency

| Attribute | Details |
| :--- | :--- |
| **Rule ID** | `AG003` |
| **Name** | Service to Controller Inverted Dependency |
| **Default Severity** | `ERROR` |
| **Category** | Inverted Layer Hierarchy |
| **Analysis Engine** | Bytecode Analysis (ArchUnit ASM) + Dependency Graph |
| **Core Classes** | `AG003ServiceToControllerRule, ArchUnitClassAnalyzer` |

---

## Executive Summary & Architectural Rationale

In clean hexagonal or layered architecture, dependencies point inward or downward: presentation depends on business services, but never the reverse.
1. **Adapter Coupling:** A service coupled to a web controller cannot be reused by scheduled cron tasks, Kafka stream processors, batch jobs, or integration tests without bringing in the web servlet context.
2. **Circular Dependencies:** Since controllers inevitably call services, a service depending on a controller immediately establishes a cyclic dependency ($A \to B \to A$), destabilizing the Spring ApplicationContext container startup.

---

## How It Is Implemented

Bytecode analysis inspects dependencies where `sourceIsService` is true and `targetIsController` is true:
1. `context.isClassInLayer(sourceClass, "SERVICE")` is resolved using `@Service` stereotype, package pattern, or class suffix.
2. `context.isClassInLayer(targetClass, "CONTROLLER")` is resolved using `@RestController`, `@Controller`, package pattern, or class suffix.
3. Any dependency vector (field, parameter, return type, invocation) triggers `AG003`.

---

### False Positive (FP) Handling & Mitigations
1. **Shared DTOs & Command Records:**
   - Services frequently exchange DTOs (e.g. `CreateOrderRequest`) with controllers. Because request/response records and DTOs are not tagged as `CONTROLLER`, these dependencies are ignored.
2. **Web Framework Interfaces:**
   - If a service implements a standard framework interface or interacts with Spring Security context holders, these framework types are not tagged as user controllers.
3. **Class Suffix Conflicts:**
   - If a class is named `ControllerAdviceService`, explicit `@Service` stereotype assigns it to `SERVICE`, but naming heuristics are prevented from classifying it as `CONTROLLER` due to Tier 1 stereotype precedence.

---

### False Negative (FN) Handling & Defense-in-Depth
1. **Parameter Passing of Controller References:**
   - If a service method takes `(UserController controller)` as an argument, ArchUnit method parameter analysis flags it even if no field injection exists.
2. **Generic & Container Types:**
   - If a service has a collection or map holding controllers (`List<BaseController>`), bytecode generic signatures are examined.
3. **Package & Suffix Fallbacks:**
   - Services without `@Service` annotations that follow standard package conventions (`com.company.service.*`) are still tracked via Tier 2 package patterns.

---

## Code Examples

### Non-Compliant (Violating Anti-Pattern)

```java
// VIOLATION: OrderService injects UserController to lookup user session
@Service
public class OrderService {

    private final UserController userController; // Inverted dependency!

    public OrderService(UserController userController) {
        this.userController = userController;
    }

    public void createOrder(String orderId) {
        var user = userController.getCurrentUser();
        // process order...
    }
}
```

### Compliant (Refactored Best Practice)

```java
// COMPLIANT: OrderService receives UserContext or domain DTO as method parameter
@Service
public class OrderService {

    public void createOrder(String orderId, UserContext userContext) {
        // Business logic uses pure domain context
    }
}
```

### Legitimate Edge Cases Handled Without Alerting

```java
// COMPLIANT: Service returning a DTO that is consumed by controllers
@Service
public class OrderService {
    public OrderDto getOrder(String id) { // OrderDto is not a Controller
        return new OrderDto(id, 100.0);
    }
}
```

---

## Configuration & Tuning

In `archguard.yml`, you can customize rule severity, exclude specific classes, or adjust thresholds:

```yaml
rules:
  AG003:
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
@SuppressArchitecture("AG003")
public void myMethod() {
    // ...
}

// Or using standard Java @SuppressWarnings
@SuppressWarnings("archguard:AG003")
public void myMethod() {
    // ...
}
```
