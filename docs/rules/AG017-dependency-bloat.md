# AG017: Injected Dependency Bloat / God Class

| Attribute | Details |
| :--- | :--- |
| **Rule ID** | `AG017` |
| **Name** | Injected Dependency Bloat / God Class |
| **Default Severity** | `WARNING` |
| **Category** | Maintainability & Modularity |
| **Analysis Engine** | AST Source Analysis (JavaParser) |
| **Core Classes** | `AG017DependencyBloatRule, CodeMetricsInspector` |

---

## Executive Summary & Architectural Rationale

Beans that inject numerous collaborators signal an unmanageable God Class:
1. **Single Responsibility Violation:** When a service requires 8, 10, or 15 collaborating repositories and services, it is doing too much and acting as an architectural bottleneck.
2. **High Change Frequency & Merge Conflicts:** God classes change for every feature request, increasing merge conflicts and defect rates.
3. **Brittle Unit Testing:** Writing unit tests requires setting up 10+ mocks, discouraging comprehensive test coverage.

---

## How It Is Implemented

AST analysis inspects fields in Spring bean components (`@Service`, `@Component`, `@RestController`):
1. Filters out static fields.
2. Counts injected collaborators:
   - Fields annotated with `@Autowired`, `@Inject`, `@Resource`.
   - Non-static `final` fields (used in constructor injection and Lombok `@RequiredArgsConstructor`).
3. If total collaborator count $\ge 7$, records `InjectedDependencyBloatFinding`.

---

### False Positive (FP) Handling & Mitigations
1. **Configuration Properties, Primitives & Inline Utilities:**
   - Fields annotated with `@Value` or `@ConfigurationProperties`, primitive/scalar types (`int`, `boolean`, `String`, `Duration`), and inline utility instantiations (`new ObjectMapper()`, `Clock`, `Random`) do not count toward collaborator dependency bloat.
2. **Explicit & Lombok Constructor Injection Support:**
   - Accurately counts parameters in explicit constructors as well as Lombok `@RequiredArgsConstructor` / `@AllArgsConstructor` beans while ignoring non-bean configuration parameters.
3. **Configurable Threshold:**
   - The threshold (default: 7) can be tuned in `archguard.yml` for legacy services undergoing refactoring.

---

### False Negative (FN) Handling & Defense-in-Depth
1. **Lombok `@RequiredArgsConstructor`:**
   - Evaluates `private final` fields, ensuring constructor-injected dependencies are counted identically to explicit `@Autowired` fields.

---

## Code Examples

### Non-Compliant (Violating Anti-Pattern)

```java
// VIOLATION: Service injecting 8 collaborators (God Class)
@Service
@RequiredArgsConstructor
public class CheckoutService {
    private final OrderRepository orderRepo;
    private final PaymentClient paymentClient;
    private final InventoryClient inventoryClient;
    private final NotificationService notificationService;
    private final AuditRepository auditRepo;
    private final TaxCalculationService taxService;
    private final ShippingService shippingService;
    private final FraudDetectionService fraudService; // 8 collaborators!
}
```

### Compliant (Refactored Best Practice)

```java
// COMPLIANT: Decomposed using Facade and Strategy patterns
@Service
@RequiredArgsConstructor
public class CheckoutService {
    private final OrderRepository orderRepo;
    private final PaymentOrchestrator paymentOrchestrator; // Aggregates payment, tax, fraud
    private final FulfillmentOrchestrator fulfillmentOrchestrator; // Aggregates shipping, inventory
}
```

### Legitimate Edge Cases Handled Without Alerting

```java
// COMPLIANT: Bean with only 4 collaborators
@Service
@RequiredArgsConstructor
public class UserService {
    private final UserRepository userRepo;
    private final PasswordEncoder passwordEncoder;
    private final EmailService emailService;
}
```

---

## Configuration & Tuning

In `archguard.yml`, you can customize rule severity, exclude specific classes, or adjust thresholds:

```yaml
rules:
  AG017:
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
@SuppressArchitecture("AG017")
public void myMethod() {
    // ...
}

// Or using standard Java @SuppressWarnings
@SuppressWarnings("archguard:AG017")
public void myMethod() {
    // ...
}
```
