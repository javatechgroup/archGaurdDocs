# AG002: Controller to Controller Direct Dependency

| Attribute | Details |
| :--- | :--- |
| **Rule ID** | `AG002` |
| **Name** | Controller to Controller Direct Dependency |
| **Default Severity** | `ERROR` |
| **Category** | Layer Boundary Enforcement |
| **Analysis Engine** | Bytecode Analysis (ArchUnit ASM) + Dependency Graph |
| **Core Classes** | `AG002ControllerToControllerRule, ArchUnitClassAnalyzer` |

---

## Executive Summary & Architectural Rationale

Web controllers are entry point adapters responsible for HTTP transport concerns (deserializing JSON, HTTP headers, cookies, query parameters, content negotiation, status codes).
1. **Coupled Presentation Logic:** Calling one controller from another tightly couples distinct HTTP endpoints. If one endpoint changes its request payload or URL mapping, other unrelated controllers break.
2. **Context Loss & Overhead:** Injected controllers do not execute through the HTTP dispatcher servlet filter chain, meaning Spring Security context, handler interceptors, and validation decorators do not run properly.
3. **Architectural Confusion:** Controllers are meant to orchestrate calls down into domain services, not horizontally across adjacent presentation adapters.

---

## How It Is Implemented

Bytecode analysis evaluates direct dependencies between classes tagged as `CONTROLLER`:
1. `ArchUnitClassAnalyzer` tags all `@RestController`, `@Controller`, and `*Controller` classes as `CONTROLLER`.
2. Direct dependencies between distinct classes (`!targetName.equals(sourceName)`) where both source and target belong to `CONTROLLER` are recorded.
3. `AG002ControllerToControllerRule` iterates over dependencies, groups by `sourceClass -> targetClass`, and generates violations.

---

### False Positive (FP) Handling & Mitigations
1. **Self-Invocation within Same Controller:**
   - Methods calling private helper methods or other endpoint handlers within the *same* controller class have `sourceName.equals(targetName)`. These are strictly filtered out by `!targetName.equals(sourceName)`.
2. **Controller Inheritance from Base Abstract Controller:**
   - Many enterprise projects share common response utilities via an abstract class (e.g. `BaseApiController`). If `BaseApiController` does not handle endpoints directly and is excluded or categorized under infrastructure/common, subclasses extending it are not misidentified as cross-calling peer controllers.
3. **Intra-Pair Aggregation:**
   - Multiple calls between Controller A and Controller B are merged into a single actionable violation.

---

### False Negative (FN) Handling & Defense-in-Depth
1. **Constructor & Field Injection:**
   - Catches controllers injected via field `@Autowired` or constructor injection.
2. **Sub-Resource Method Returns:**
   - If `OrderController` returns `PaymentController` (e.g., JAX-RS / Spring sub-resource pattern), bytecode method return types capture `PaymentController` as a target dependency.
3. **Static Method Utility Calls:**
   - If `OrderController` calls a static utility method on `UserController.formatUserResponse(...)`, the bytecode `JavaMethodCall` captures the cross-controller reference.

---

## Code Examples

### Non-Compliant (Violating Anti-Pattern)

```java
// VIOLATION: OrderController injects and calls PaymentController directly
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    private final PaymentController paymentController; // Cross-controller injection!

    public OrderController(PaymentController paymentController) {
        this.paymentController = paymentController;
    }

    @PostMapping("/{orderId}/checkout")
    public ResponseEntity<Void> checkout(@PathVariable String orderId) {
        paymentController.processPayment(orderId); // Bypasses service layer!
        return ResponseEntity.ok().build();
    }
}
```

### Compliant (Refactored Best Practice)

```java
// COMPLIANT: Both controllers share PaymentService in the domain layer
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    private final PaymentService paymentService; // Shared domain service

    public OrderController(PaymentService paymentService) {
        this.paymentService = paymentService;
    }

    @PostMapping("/{orderId}/checkout")
    public ResponseEntity<Void> checkout(@PathVariable String orderId) {
        paymentService.processPayment(orderId);
        return ResponseEntity.ok().build();
    }
}
```

### Legitimate Edge Cases Handled Without Alerting

```java
// COMPLIANT: Internal helper method call within the same controller
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @GetMapping("/{id}")
    public OrderResponse getOrder(@PathVariable String id) {
        return buildResponse(id); // Self-call: ignored by AG002
    }

    private OrderResponse buildResponse(String id) {
        return new OrderResponse(id, "PROCESSED");
    }
}
```

---

## Configuration & Tuning

In `archguard.yml`, you can customize rule severity, exclude specific classes, or adjust thresholds:

```yaml
rules:
  AG002:
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
@SuppressArchitecture("AG002")
public void myMethod() {
    // ...
}

// Or using standard Java @SuppressWarnings
@SuppressWarnings("archguard:AG002")
public void myMethod() {
    // ...
}
```
