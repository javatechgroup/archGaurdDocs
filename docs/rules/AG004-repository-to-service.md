# AG004: Repository to Service Inverted Dependency

| Attribute | Details |
| :--- | :--- |
| **Rule ID** | `AG004` |
| **Name** | Repository to Service Inverted Dependency |
| **Default Severity** | `ERROR` |
| **Category** | Inverted Layer Hierarchy |
| **Analysis Engine** | Bytecode Analysis (ArchUnit ASM) + Dependency Graph |
| **Core Classes** | `AG004RepositoryToServiceRule, ArchUnitClassAnalyzer` |

---

## Executive Summary & Architectural Rationale

The repository/persistence layer represents the bottom-most layer in classical enterprise architecture.
1. **Circular Call Chains:** Services orchestrate business flows and call repositories to fetch and save state. If a repository calls back into a service, circular dependency graphs inevitably emerge.
2. **Transaction Scope Corruption:** Repositories executing business service calls can inadvertently trigger new transactions, nested rollbacks, or premature session flushes.
3. **Separation of Concerns:** Repositories should only translate domain entities to database persistence records; they must have zero awareness of higher-level business workflows.

---

## How It Is Implemented

Bytecode analysis inspects dependencies where `sourceIsRepo` is true and `targetIsService` is true:
1. `context.isClassInLayer(sourceClass, "REPOSITORY")` resolves repositories via `@Repository`, `isAssignableToRepository`, or package/naming heuristics.
2. `context.isClassInLayer(targetClass, "SERVICE")` resolves services via `@Service` or package/naming heuristics.
3. Any cross-layer reference triggers `AG004`.

---

### False Positive (FP) Handling & Mitigations
1. **Repository Custom Implementation Interfaces:**
   - Spring Data allows custom implementations (e.g. `UserRepositoryCustom` and `UserRepositoryImpl`). These are recognized as persistence components and not mistaken for services.
2. **Domain Model & Entity References:**
   - Repositories referencing JPA Entities, Enums, or Embeddables are not flagged because entities are not in the `SERVICE` layer.
3. **Spring Data Utility Services:**
   - Standard Spring framework classes (e.g. `EntityManager`, `JdbcTemplate`, `TransactionTemplate`) are excluded from project class comparisons.

---

### False Negative (FN) Handling & Defense-in-Depth
1. **Custom Repository Impl (`*RepositoryImpl`):**
   - Custom implementations that inject business services to perform complex business checks before running a query are fully inspected by bytecode scanning.
2. **Spring Data Repository extending Repository:**
   - Even if the interface lacks `@Repository`, `isAssignableToRepository` identifies it and ensures inverted calls are caught.

---

## Code Examples

### Non-Compliant (Violating Anti-Pattern)

```java
// VIOLATION: Repository implementation injects AuditService
@Repository
public class CustomUserRepositoryImpl implements CustomUserRepository {

    private final AuditService auditService; // Inverted dependency!

    public CustomUserRepositoryImpl(AuditService auditService) {
        this.auditService = auditService;
    }

    @Override
    public User findUserWithAudit(Long id) {
        auditService.recordAccess(id); // Persistence layer calling service!
        return null;
    }
}
```

### Compliant (Refactored Best Practice)

```java
// COMPLIANT: AuditService or UserService coordinates the repository and auditing
@Service
public class UserService {

    private final UserRepository userRepository;
    private final AuditService auditService;

    public UserService(UserRepository userRepository, AuditService auditService) {
        this.userRepository = userRepository;
        this.auditService = auditService;
    }

    public User getUser(Long id) {
        auditService.recordAccess(id);
        return userRepository.findById(id).orElseThrow();
    }
}
```

### Legitimate Edge Cases Handled Without Alerting

```java
// COMPLIANT: Repository using JPA EntityManager directly
@Repository
public class OrderRepositoryImpl {
    @PersistenceContext
    private EntityManager entityManager; // Compliant framework dependency
}
```

---

## Configuration & Tuning

In `archguard.yml`, you can customize rule severity, exclude specific classes, or adjust thresholds:

```yaml
rules:
  AG004:
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
@SuppressArchitecture("AG004")
public void myMethod() {
    // ...
}

// Or using standard Java @SuppressWarnings
@SuppressWarnings("archguard:AG004")
public void myMethod() {
    // ...
}
```
