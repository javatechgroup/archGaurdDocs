# AG006: Package Boundary Violation

| Attribute | Details |
| :--- | :--- |
| **Rule ID** | `AG006` |
| **Name** | Package Boundary Violation |
| **Default Severity** | `WARNING` |
| **Category** | Custom Architectural Boundaries |
| **Analysis Engine** | Bytecode Analysis (ArchUnit ASM) + Pattern Matching |
| **Core Classes** | `AG006PackageBoundaryRule, RuleSuppressionFilter` |

---

## Executive Summary & Architectural Rationale

In modular monoliths and domain-driven designs (DDD):
1. **Domain Boundary Erosion:** Modules (e.g. `billing`, `shipping`, `inventory`) must only interact through explicit public APIs or interfaces. Direct internal coupling across domain boundaries erodes modularity.
2. **Uncontrolled Imports:** Without boundary enforcement, developers inadvertently import internal repository or entity classes from adjacent modules, preventing future microservice extraction.
3. **Configurable Governance:** Allows teams to specify architectural guardrails (e.g. `com.app.core.*` can only depend on `com.app.common.*`).

---

## How It Is Implemented

Evaluates project dependencies against `packageBoundaries` in `archguard.yml`:
1. Reads `context.getConfig().getPackageBoundaries()` mapping source package patterns to allowed target package patterns.
2. Iterates over all bytecode `DependencyInfo` records.
3. Checks if `sourcePkg` matches a configured boundary rule key. If matched, verifies whether `targetPkg` matches any allowed target pattern via `RuleSuppressionFilter.matchesPattern()`.
4. If not allowed, records a violation with full evidence and source line numbers.

---

### False Positive (FP) Handling & Mitigations
1. **Intra-Package References:**
   - Classes calling other classes within the **exact same package** (`sourcePkg.equals(targetPkg)`) are always permitted automatically without needing to declare the package in its own allowed list.
2. **Third-Party & Standard Libraries:**
   - Dependencies targeting `java.*`, `javax.*`, `jakarta.*`, `org.springframework.*`, or external dependencies are not project classes and do not trigger project boundary violations.
3. **Ant-Style Wildcards:**
   - Pattern matching supports `*` (single subpackage) and `**` (recursive subpackages), preventing false positives from package hierarchy variations.

---

### False Negative (FN) Handling & Defense-in-Depth
1. **Bytecode-Level Method & Field Calls:**
   - Catches references even if written without `import` statements (e.g. using fully qualified class names `com.mycompany.internal.SecretUtil.run()`).
2. **Inheritance & Interfaces Across Packages:**
   - Extending a class or implementing an interface from a forbidden package is detected via bytecode superclass and interface tables.

---

## Code Examples

### Non-Compliant (Violating Anti-Pattern)

```java
// VIOLATION: Order module directly accesses internal Billing repository
package com.mycompany.order.service;

import com.mycompany.billing.internal.BillingRepository; // FORBIDDEN by packageBoundaries!

public class OrderCheckoutService {
    private BillingRepository billingRepo;
}
```

### Compliant (Refactored Best Practice)

```java
// COMPLIANT: Order module accesses Billing via its public API interface
package com.mycompany.order.service;

import com.mycompany.billing.api.BillingService; // Allowed public boundary

public class OrderCheckoutService {
    private final BillingService billingService;

    public OrderCheckoutService(BillingService billingService) {
        this.billingService = billingService;
    }
}
```

### Legitimate Edge Cases Handled Without Alerting

```java
// COMPLIANT: Intra-package access without explicit boundary listing
package com.mycompany.order.service;

public class OrderPricingHelper {
    // Calling OrderCheckoutService in same package: ALWAYS ALLOWED
}
```

---

## Configuration & Tuning

In `archguard.yml`, you can customize rule severity, exclude specific classes, or adjust thresholds:

```yaml
rules:
  AG006:
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
@SuppressArchitecture("AG006")
public void myMethod() {
    // ...
}

// Or using standard Java @SuppressWarnings
@SuppressWarnings("archguard:AG006")
public void myMethod() {
    // ...
}
```
