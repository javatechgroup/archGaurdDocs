# AG007: Prohibited Field Injection

| Attribute | Details |
| :--- | :--- |
| **Rule ID** | `AG007` |
| **Name** | Prohibited Field Injection |
| **Default Severity** | `WARNING` |
| **Category** | Clean Code & Testability |
| **Analysis Engine** | AST Source Analysis (JavaParser) |
| **Core Classes** | `AG007FieldInjectionRule, InjectionAndStateInspector` |

---

## Executive Summary & Architectural Rationale

Field injection using `@Autowired` or `@Inject` on private fields is widely regarded as a major anti-pattern in modern Spring development:
1. **Hides Dependencies:** Classes using field injection hide their collaborating dependencies from consumers and constructors.
2. **Prevents Immutability:** Injected fields cannot be marked `final`, leaving bean instances susceptible to post-initialization mutation.
3. **Severe Unit Testing Pain:** In order to unit test a field-injected bean, developers must resort to reflection utilities (`ReflectionTestUtils.setField()`) or slow Spring test runners (`@SpringBootTest`), drastically slowing CI pipelines.
4. **NPE Risks during Manual Instantiation:** Calling `new OrderService()` in a test leaves all dependencies `null`, throwing immediate `NullPointerException`.

---

## How It Is Implemented

AST analysis inspects all `FieldDeclaration` nodes in JavaParser:
1. Iterates over field annotations:
   - Matches `@Autowired`, `@Inject`, `@Resource`.
   - Resolves both simple names (`@Autowired`) and fully qualified package annotations (`@org.springframework.beans.factory.annotation.Autowired`, `@jakarta.inject.Inject`, `@javax.annotation.Resource`).
2. Records `SourceFieldInjectionFinding` for each variable declarator with exact line number and source snippet.

---

### False Positive (FP) Handling & Mitigations
1. **Lombok `@RequiredArgsConstructor` & Constructor Injection:**
   - Constructor-injected fields are marked `private final` and do **not** carry annotations on the field declaration. AG007 does not flag final fields.
2. **Setter Injection:**
   - `@Autowired` placed on a setter method (`@Autowired public void setRepo(...)`) is method-level injection and is not flagged by AG007.
3. **Configuration Properties & Value Injection:**
   - Fields annotated with `@Value("${property}")` or `@PersistenceContext` are not dependency injection beans and are excluded.

---

### False Negative (FN) Handling & Defense-in-Depth
1. **Multi-Variable Field Declarations:**
   - If written as `@Autowired private OrderRepo r1, r2;`, JavaParser's `field.getVariables()` extracts **all** declared variables on that field node.
2. **Jakarta & JavaEE Aliases:**
   - Checks `jakarta.inject.Inject`, `javax.inject.Inject`, `jakarta.annotation.Resource`, `javax.annotation.Resource`, and Spring `@Autowired`.

---

## Code Examples

### Non-Compliant (Violating Anti-Pattern)

```java
// VIOLATION: Field injection on private fields
@Service
public class OrderService {

    @Autowired // Anti-pattern: hidden dependency, mutable field
    private OrderRepository orderRepository;

    @Inject
    private PaymentClient paymentClient;
}
```

### Compliant (Refactored Best Practice)

```java
// COMPLIANT: Constructor injection with final immutable fields
@Service
public class OrderService {

    private final OrderRepository orderRepository;
    private final PaymentClient paymentClient;

    public OrderService(OrderRepository orderRepository, PaymentClient paymentClient) {
        this.orderRepository = orderRepository;
        this.paymentClient = paymentClient;
    }
}

// ALSO COMPLIANT: Using Lombok @RequiredArgsConstructor
@Service
@RequiredArgsConstructor
public class UserService {
    private final UserRepository userRepository; // Clean, final, and immutable
}
```

### Legitimate Edge Cases Handled Without Alerting

```java
// COMPLIANT: @Value property injection is not bean collaborator injection
@Service
public class ReportService {
    @Value("${report.timeout:30}")
    private int timeoutSeconds; // Permitted: configuration property injection
}
```

---

## Configuration & Tuning

In `archguard.yml`, you can customize rule severity, exclude specific classes, or adjust thresholds:

```yaml
rules:
  AG007:
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
@SuppressArchitecture("AG007")
public void myMethod() {
    // ...
}

// Or using standard Java @SuppressWarnings
@SuppressWarnings("archguard:AG007")
public void myMethod() {
    // ...
}
```
