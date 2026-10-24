# Configuration Guide

Architecture Guard can be configured via `archguard.yml` in the target project root, or inline via Maven plugin configuration.

---

## Full `archguard.yml` Example

```yaml
analysis:
  failOnSeverity: ERROR          # Levels: INFO, WARNING, ERROR, NONE
  failOnNewViolations: true      # When using baseline comparison
  threads: 4                     # Concurrency: 0/omitted = all CPU cores, 1 = sequential debug
  outputDir: target/architecture-guard
  baselineFile: archguard-baseline.json

rules:
  AG001:
    enabled: true
    severity: ERROR
    exclusions:
      - com.mycompany.app.legacy.*
  AG005:
    enabled: true
    severity: ERROR
  AG006:
    enabled: true
    severity: WARNING
  AG007:
    enabled: true
    severity: WARNING

packages:
  controller:
    - com.mycompany.app.controller.*
    - com.mycompany.app.web.*
  service:
    - com.mycompany.app.service.*
  repository:
    - com.mycompany.app.repository.*
  entity:
    - com.mycompany.app.entity.*

packageBoundaries:
  com.mycompany.app.controller:
    - com.mycompany.app.service
    - com.mycompany.app.dto
  com.mycompany.app.service:
    - com.mycompany.app.repository
    - com.mycompany.app.dto
  com.mycompany.app.repository:
    - com.mycompany.app.entity

exclusions:
  - com.mycompany.app.generated.*
  - com.mycompany.app.dto.*

reportFormats:
  - CONSOLE
  - JSON
  - HTML
```

---

## Centralized Concurrency & Multi-Thread Sizing

ArchGuard executes with an isolated, dedicated worker thread pool (`ArchGuardTaskExecutor`):

> [!IMPORTANT]
> **Rule of Thumb: Do not configure `threads` in POM or YAML files.**
> ArchGuard performs purely in-memory CPU computation (AST parsing, graph building, rule evaluation) with zero I/O blocking. Therefore, the optimal thread count mathematically equals your physical/logical CPU core count ($\text{Threads} = \text{Cores}$).
> - Omitting the configuration lets ArchGuard automatically detect all available cores via `Runtime.getRuntime().availableProcessors()`.
> - Do not specify thread counts higher than your CPU cores, as this triggers context switching and cache invalidation.
> - Avoid hardcoding `<threads>` in `pom.xml`, which blocks dynamic CLI overrides like `-Darchguard.threads=1`.

| Sizing Value | Behavior | Ideal Environment |
| :--- | :--- | :--- |
| **Omitted / `0` (Recommended)** | **Auto-Detect (Default)**: Uses `Runtime.getRuntime().availableProcessors()` | Standard developer builds & dedicated CI runners. |
| **`1`** | **Deterministic Sequential Mode**: Runs directly on caller thread; disables thread pools | Local IDE debugging, setting breakpoints, isolating race conditions. |
| **Specific `N`** (e.g. `2`, `4`) | **Fixed Worker Pool**: Allocates exactly `N` worker threads (`archguard-worker-*`) | Budget CI containers with vCPU limits (e.g. 2 vCPU pods) to avoid CPU throttling. |

### CLI Override
You can override thread settings dynamically without modifying configuration files:
```bash
mvn clean verify -Darchguard.threads=4
```

---

## Safe Adoption in Existing (Legacy) Projects

To introduce Architecture Guard into a legacy application with existing technical debt without breaking existing CI builds:

1. **Step 1**: Run `mvn architecture-guard:baseline` to capture all pre-existing debt in `.architecture-guard-baseline.json`.
2. **Step 2**: Ensure `failOnNewViolationsOnly: true` is configured in `pom.xml` or `archguard.yml`.
3. **Step 3**: Any *new* violation introduced by ongoing PRs will fail the build, while existing debt is cataloged.

---

## Multi-Module Upward Discovery

In Maven multi-module aggregator projects:
- If a submodule does not define its own `archguard.yml`, ArchGuard **automatically traverses up the directory tree (up to 4 parent levels)** to discover the root configuration file.
- Aggregator and parent POM projects (`<packaging>pom</packaging>`) are automatically skipped cleanly during reactor builds.

---

## In-Code Suppressions

Suppress specific architectural violations directly in Java code without failing the build. Suppressed issues remain visible in the HTML report under the `🛡️ Suppressed` tab for auditing.

### Method-Level Suppression (Recommended)
```java
// Suppress specific rules on a method
@SuppressWarnings({"archguard:AG018", "archguard:AG020"})
public void processBatch() { ... }

// Suppress all ArchGuard rules on a method
@SuppressWarnings("archguard")
public void legacyMethod() { ... }
```

### Class-Level Suppression
```java
// Suppress a rule across the entire class
@SuppressWarnings("archguard:AG001")
public class LegacyController { ... }

// Dedicated annotation with audit rationale
@SuppressArchitecture(rule = "AG001", reason = "Refactoring planned in next sprint")
public class LegacyController { ... }
```
