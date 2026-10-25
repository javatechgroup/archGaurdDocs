# ArchGuard — Developer & User Manual

> **Automated Architecture Governance & Code Quality Gate for Modern Java & Spring Boot Applications.**

---

## Table of Contents
1. [Overview](#1-overview)
2. [Prerequisites & System Requirements](#2-prerequisites--system-requirements)
3. [Quick Start Guide](#3-quick-start-guide)
4. [Build Tool Configuration (Maven & Gradle)](#4-build-tool-configuration-maven--gradle)
5. [Core Commands & Execution](#5-core-commands--execution)
6. [Baseline & Technical Debt Ratcheting](#6-baseline--technical-debt-ratcheting)
7. [Comprehensive Rule Catalog (AG001 – AG033)](#7-comprehensive-rule-catalog-ag001--ag033)
8. [In-Code Rule Suppressions](#8-in-code-rule-suppressions)
9. [Configuration File Reference (`archguard.yml`)](#9-configuration-file-reference-archguardyml)
10. [Reports & Diagnostics](#10-reports-diagnostics--automated-pr-code-review)
11. [CI/CD Pipeline Integration](#11-cicd-pipeline-integration)
12. [Operational Best Practices & Crash Prevention](#12-operational-best-practices--crash-prevention)
13. [Troubleshooting & FAQ](#13-troubleshooting--faq)

---

## 1. Overview

**ArchGuard** (available as the Apache Maven plugin `io.github.javatechgroup:architecture-guard-maven-plugin` and Gradle plugin `io.github.javatechgroup.architecture-guard`) is a static analysis and quality gate tool designed specifically for Java 21+ and Spring Boot applications.

Unlike traditional linters that only inspect syntax or basic code smells, ArchGuard combines:
- **High-performance Bytecode Analysis (ASM):** Inspects compiled class files to analyze type hierarchies, method calls, field types, and class-level layer dependencies.
- **Precision AST Source Inspection (JavaParser):** Scans source code for semantic smells such as direct entity leakage in controller APIs, mutable static collections, field-level `@Autowired` injections, swallowed exception blocks, and hardcoded credentials.
- **Non-Intrusive Quality Gates with Baselining:** Protects existing legacy projects from developer frustration by freezing existing violations in a baseline snapshot file and only failing the build on **newly introduced** violations.

```mermaid
flowchart TD
    classDef inputNode fill:#e0f2fe,stroke:#0284c7,stroke-width:2px,color:#0369a1;
    classDef engineNode fill:#f0fdf4,stroke:#16a34a,stroke-width:2px,color:#15803d;
    classDef govNode fill:#fef3c7,stroke:#d97706,stroke-width:2px,color:#b45309;
    classDef reportNode fill:#f3e8ff,stroke:#9333ea,stroke-width:2px,color:#7e22ce;
    classDef passNode fill:#dcfce7,stroke:#16a34a,stroke-width:2px,color:#15803d;
    classDef failNode fill:#fee2e2,stroke:#dc2626,stroke-width:2px,color:#991b1b;

    A["Java 21 Source (.java)"]:::inputNode -->|javac| B["Compiled Classes (.class)"]:::inputNode
    A --> C["AST Source Analyzer (JavaParser)"]:::engineNode
    B --> D["Bytecode Analyzer (ASM)"]:::engineNode
    C --> E["ArchGuard Engine"]:::engineNode
    D --> E
    E --> F{"Baseline Snapshot (.json) Present?"}:::govNode
    F -->|Yes| G["Multiset Differential Engine\n(NEW vs BASELINED vs RESOLVED)"]:::govNode
    F -->|No| H["All Violations Evaluated"]:::govNode
    G --> I{"New Violations >= failOnSeverity?"}:::govNode
    H --> J{"Total Violations >= failOnSeverity?"}:::govNode
    I -->|Yes| K["BUILD FAILURE ❌"]:::failNode
    I -->|No| L["BUILD SUCCESS ✅"]:::passNode
    J -->|Yes| K
    J -->|No| L
    G --> M["HTML / JSON / Console Reports"]:::reportNode
    H --> M
```

---

## 2. Prerequisites & System Requirements

- **JDK:** Java 21 or higher
- **Application Language:** Java (`.java` source files). *(Note: Kotlin application source files `.kt` are not supported; "Kotlin DSL" refers strictly to Gradle build scripts `build.gradle.kts`)*
- **Build Tools:** Apache Maven 3.8.0+ or Gradle 8.0+
- **Frameworks Supported:** Spring Boot 3.x, Spring Framework 6.x, Standard Java applications

> [!TIP]
> **Memory Sizing Notice:** For standard microservices, default Maven/Gradle memory or `-Xmx1g` is plenty. For very large codebases (2,500+ classes), scale up to `-Xmx2g` (or `-Xmx4g` on mega-monoliths) to prevent `OutOfMemoryError`. Always ensure the heap limit does not exceed your CI runner's total container RAM. See [Section 12: Operational Best Practices & Crash Prevention](#12-operational-best-practices--crash-prevention) for the sizing matrix.

---

## 3. Quick Start Guide

ArchGuard operates on a **Zero-Configuration** philosophy: once declared in your build (`pom.xml` or `build.gradle`), it evaluates all 33 rules, auto-detects Spring/JPA layers, and generates HTML & SARIF reports without requiring any external configuration files.

### Step 1: Add ArchGuard to Your Build (Required)

#### Option A: Maven (`pom.xml`)
Add `architecture-guard-maven-plugin` to your project's `<build><plugins>` section:

```xml
<build>
    <plugins>
        <plugin>
            <groupId>io.github.javatechgroup</groupId>
            <artifactId>architecture-guard-maven-plugin</artifactId>
            <version>1.0.5</version>
            <executions>
                <execution>
                    <goals>
                        <goal>check</goal>
                    </goals>
                </execution>
            </executions>
            <configuration>
                <!-- Default failure threshold is ERROR. Set to WARNING to fail on all code smells -->
                <failOnSeverity>ERROR</failOnSeverity>
                <outputDirectory>${project.build.directory}/architecture-guard</outputDirectory>
            </configuration>
        </plugin>
    </plugins>
</build>
```

#### Option B: Gradle Kotlin DSL (`build.gradle.kts`)
Add the plugin ID and optional configuration block:

```kotlin
plugins {
    id("io.github.javatechgroup.architecture-guard") version "1.0.5"
}

archguard {
    // Default failure threshold is ERROR. Set to WARNING to fail on all code smells
    failOnSeverity.set("ERROR")
}
```

#### Option C: Gradle Groovy DSL (`build.gradle`)
```groovy
plugins {
    id 'io.github.javatechgroup.architecture-guard' version '1.0.5'
}

archguard {
    failOnSeverity = 'ERROR'
}
```

### Step 2: (For Existing Projects) Freeze Legacy Debt

If adopting ArchGuard on an existing project, freeze pre-existing violations into a baseline so legacy code never blocks your build:

**Maven:**
```bash
mvn architecture-guard:baseline
```

**Gradle:**
```bash
./gradlew archGuardBaseline
```
*(Creates `.architecture-guard-baseline.json`. Only newly introduced violations in upcoming pull requests will fail the build).*

### Step 3: Run Analysis & Enforce Quality Gate

**Maven:**
```bash
mvn clean verify
```

**Gradle:**
```bash
./gradlew check
```

ArchGuard will compile your classes, evaluate all 33 architecture and quality rules, generate offline interactive HTML, JSON, and standard SARIF 2.1.0 reports in `target/architecture-guard/` (Maven) or `build/reports/architecture-guard/` (Gradle), and fail the build if violations at or above the threshold are detected.

---

### 💡 (Optional) Scaffold Custom Configuration (`archguard.yml`)

If your project requires non-standard layer package patterns (e.g., Hexagonal ports & adapters), customized rule severities, or package boundary policies, generate a starter configuration file using:

**Maven:**
```bash
mvn architecture-guard:init
```

**Gradle:**
```bash
./gradlew archGuardInit
```
*(Auto-detects base packages and writes a commented `archguard.yml` in your project root).*

---

## 4. Build Tool Configuration (Maven & Gradle)

### 4.1 Maven Plugin Configuration

#### Full `<configuration>` Options

| Parameter | Type | Default | CLI Property | Description |
| :--- | :--- | :--- | :--- | :--- |
| `failOnSeverity` | `String` | `ERROR` | `-Darchguard.failOnSeverity` | Minimum severity that triggers a build failure (`ERROR`, `WARNING`, `INFO`, `NONE`). |
| `threads` | `Integer` | (auto-detect) | `-Darchguard.threads` | Worker thread count for parallel analysis (0 or omitted = auto-detect all CPU cores; 1 = single-threaded sequential debugging). |
| `failOnNewViolationsOnly` | `boolean` | `true` | `-Darchguard.failOnNewViolationsOnly` | When a baseline file exists, only fail the build if new violations are introduced. |
| `ratchet` | `boolean` | `false` | `-Darchguard.ratchet` | Automatically update baseline file when legacy debt is resolved (ratchet mechanism). |
| `baselineFile` | `File` | `${project.basedir}/.architecture-guard-baseline.json` | `-Darchguard.baselineFile` | Custom location for the baseline snapshot file. |
| `configFile` | `File` | (auto-discovered) | `-Darchguard.configFile` | Path to custom `archguard.yml` or `architecture-guard.json`. |
| `outputDirectory` | `File` | `${project.build.directory}/architecture-guard` | `-Darchguard.outputDirectory` | Target directory for generated HTML, JSON, and SARIF reports. |
| `skip` | `boolean` | `false` | `-Darchguard.skip` | Skips ArchGuard analysis execution. |

### 4.2 Gradle Plugin Configuration

The ArchGuard Gradle plugin (`io.github.javatechgroup.architecture-guard`) registers the `archguard` configuration extension and automatically wires verification into Gradle's standard `check` task lifecycle.

#### Full `archguard { ... }` DSL Options

```kotlin
// build.gradle.kts (Kotlin DSL)
archguard {
    failOnSeverity.set("ERROR")             // ERROR, WARNING, INFO, NONE
    failOnNewViolationsOnly.set(true)       // Only fail on new violations if baseline exists
    ratchet.set(false)                      // Auto-shrink baseline when debt is fixed
    configFile.set(file("archguard.yml"))   // Optional custom configuration path
    baselineFile.set(file(".architecture-guard-baseline.json"))
    outputDirectory.set(layout.buildDirectory.dir("reports/architecture-guard"))
    skip.set(false)
}
```

```groovy
// build.gradle (Groovy DSL)
archguard {
    failOnSeverity = 'ERROR'
    failOnNewViolationsOnly = true
    ratchet = false
    configFile = file('archguard.yml')
    baselineFile = file('.architecture-guard-baseline.json')
    outputDirectory = layout.buildDirectory.dir('reports/architecture-guard')
    skip = false
}
```

| Property | Type | Default | CLI / System Property | Description |
| :--- | :--- | :--- | :--- | :--- |
| `failOnSeverity` | `Property<String>` | `ERROR` | `-Darchguard.failOnSeverity` | Minimum severity triggering build failure (`ERROR`, `WARNING`, `INFO`, `NONE`). |
| `threads` | `Property<Integer>` | (auto-detect) | `-Darchguard.threads` | Worker thread count for parallel analysis (0 or omitted = CPU cores). |
| `failOnNewViolationsOnly` | `Property<Boolean>` | `true` | `-Darchguard.failOnNewViolationsOnly` | When a baseline file exists, only fail the build on new violations. |
| `ratchet` | `Property<Boolean>` | `false` | `--ratchet` or `-Darchguard.ratchet` | Automatically update baseline file when legacy debt is resolved. |
| `baselineFile` | `RegularFileProperty` | `<rootDir>/.architecture-guard-baseline.json` | `-Darchguard.baselineFile` | Custom baseline snapshot file location. |
| `configFile` | `RegularFileProperty` | (auto-discovered) | `-Darchguard.configFile` | Path to custom `archguard.yml` or `architecture-guard.json`. |
| `outputDirectory` | `DirectoryProperty` | `<buildDir>/reports/architecture-guard` | `-Darchguard.outputDirectory` | Target directory for generated HTML, JSON, and SARIF reports. |
| `skip` | `Property<Boolean>` | `false` | `-Darchguard.skip` | Skips ArchGuard analysis execution. |

#### Registered Gradle Tasks

| Task Name | Task Type | Description |
| :--- | :--- | :--- |
| `archguard` | `ArchitectureGuardCheckTask` | Analyzes compiled classes & source files. Automatically executes during `./gradlew check`. |
| `archGuardBaseline` | `ArchitectureGuardBaselineTask` | Freezes all existing violations into `.architecture-guard-baseline.json`. |
| `archGuardInit` | `ArchitectureGuardInitTask` | Scaffolds a starter `archguard.yml` by auto-detecting root packages. |

> [!TIP]
> **Rule of Thumb on Concurrency (`threads`):**
> In almost all projects, **do not configure `<threads>` in `pom.xml` or `threads.set(...)` in `build.gradle`**.
> - **Pure In-Memory Computing:** ArchGuard operates strictly in RAM (AST syntax trees, graph traversals, and rule evaluation) with zero I/O wait. Because worker threads execute CPU instructions continuously without blocking, maximum throughput is achieved when **worker threads = number of CPU cores**. Setting threads higher than CPU cores causes CPU cache thrashing and context-switching overhead.
> - **Automatic Optimal Sizing:** When omitted, ArchGuard automatically queries `Runtime.getRuntime().availableProcessors()` to match all available CPU cores (e.g., 8 threads on an 8-core workstation, 4 threads on a 4-core CI runner).
> - **Preserves CLI Overrides:** Hardcoding threads in build files locks configuration and ignores command-line `-Darchguard.threads=...` switches. Leaving it unconfigured allows you to run single-threaded debugging (`-Darchguard.threads=1`) on demand without altering source files.

---

## 5. Core Commands & Execution

### 1. Run Standard Quality Gate
Runs the analysis during the standard build verification phase (recommended in CI):

**Maven:**
```bash
mvn clean verify
```
*(Or run the goal directly: `mvn clean compile architecture-guard:check`)*

**Gradle:**
```bash
./gradlew check
```
*(Or run the task directly: `./gradlew archguard`)*

### 2. Generate Baseline Snapshot for Legacy Projects
Freezes all current project debt into `.architecture-guard-baseline.json`:

**Maven:**
```bash
mvn architecture-guard:baseline
```

**Gradle:**
```bash
./gradlew archGuardBaseline
```

### 3. Run with Ratchet Mode Enabled
Ensures the baseline shrinks automatically as developers fix old violations:

**Maven:**
```bash
mvn clean verify -Darchguard.ratchet=true
```

**Gradle:**
```bash
./gradlew check --ratchet
# or:
./gradlew check -Darchguard.ratchet=true
```

### 4. (Optional) Scaffold Starter Configuration
Auto-detects base packages and creates a commented `archguard.yml`:

**Maven:**
```bash
mvn architecture-guard:init
```

**Gradle:**
```bash
./gradlew archGuardInit
```

### 5. Multi-Module & Multi-Project Support
ArchGuard natively supports enterprise multi-module and multi-project structures:
- **Maven Aggregators:** Parent and aggregator modules (`<packaging>pom</packaging>`) are automatically skipped with an informational log (`Skipping pom packaging module`) without raising false warnings about missing `target/classes`. Running `mvn clean verify` at root evaluates each artifact submodule cleanly.
- **Gradle Multi-Project:** In root `build.gradle.kts`, apply the plugin to Java subprojects:
  ```kotlin
  subprojects {
      apply(plugin = "java")
      apply(plugin = "io.github.javatechgroup.architecture-guard")
  }
  ```
  Running `./gradlew check` executes `archguard` across all subprojects, generating per-project HTML & SARIF reports.

### 6. Enforce Strict Warning Failures
Fails the build if any new `WARNING` (such as `System.out.println` or field `@Autowired`) is added:

**Maven:**
```bash
mvn clean verify -Darchguard.failOnSeverity=WARNING
```

**Gradle:**
```bash
./gradlew check -Darchguard.failOnSeverity=WARNING
```

---

## 6. Baseline & Technical Debt Ratcheting

### The Problem: Legacy Code Fatigue
When introducing architectural tooling into large existing projects with thousands of pre-existing violations, developers get blocked and frustrated by legacy violations they did not write.

### The Solution: Quality Gate via Differential Baselining
ArchGuard allows tech leads or CI pipelines to freeze legacy debt:

**Maven:**
```bash
mvn architecture-guard:baseline
```

**Gradle:**
```bash
./gradlew archGuardBaseline
```

This generates a version-controlled `.architecture-guard-baseline.json` file containing SHA-256 fingerprints of every current violation.

#### How It Works:
1. **Existing violations** are tagged as `BASELINED` in reports and **do not fail the build**.
2. **New violations** introduced in PRs/branches are tagged as `NEW` and **fail the build**.
3. **Fixed violations** are counted as `RESOLVED`.
4. **Multiset Protection:** If a class already has 1 baselined `System.out.println`, adding a second `System.out.println` is correctly detected as `NEW` (frequency tracking).

#### Automatic Ratchet Mode:
Enable `<ratchet>true</ratchet>` or `-Darchguard.ratchet=true`. Whenever a developer fixes a legacy violation, ArchGuard automatically removes it from `.architecture-guard-baseline.json`. Debt can only decrease, never increase.

---

## 7. Comprehensive Rule Catalog (AG001 – AG033)

### [AG001] Controller to Repository Direct Dependency
- **Severity:** `ERROR`
- **Rationale:** Web controllers must not bypass the business service layer to directly access repositories. Bypassing causes transaction leaks, duplicate business logic, and tight coupling.
- ❌ **Violation:**
  ```java
  @RestController
  public class OrderController {
      @Autowired
      private OrderRepository orderRepository; // Direct DB access from Controller!
  }
  ```
- ✅ **Remediation:**
  ```java
  @RestController
  public class OrderController {
      private final OrderService orderService;
      public OrderController(OrderService orderService) {
          this.orderService = orderService;
      }
  }
  ```

---

### [AG002] Repository to Service Inverted Dependency
- **Severity:** `ERROR`
- **Rationale:** Persistence layer components must remain low-level data adapters and never depend on high-level service orchestration classes.

---

### [AG003] Domain Model Outer Layer Dependency
- **Severity:** `ERROR`
- **Rationale:** Core domain entities and domain services must remain pure business logic and never depend on web, infrastructure, or controller components.

---

### [AG004] Circular Dependencies
- **Severity:** `ERROR`
- **Rationale:** Package-level and class-level cyclic dependencies (e.g., Service A $\rightarrow$ Service B $\rightarrow$ Service A) create tight coupling, fragile refactoring, and Spring context initialization deadlocks.
- ✅ **Remediation:** Extract shared logic into a common domain service or introduce an event-driven decoupled architecture.

---

### [AG005] Package Boundary Cross-Access
- **Severity:** `ERROR`
- **Rationale:** Enforces modular boundaries between bounded contexts (e.g., `com.example.billing` cannot directly access internal classes of `com.example.inventory`).

---

### [AG006] Spring Component in Domain Package
- **Severity:** `WARNING`
- **Rationale:** Framework-specific annotations (e.g., `@Service`, `@Component`, `@RestController`) should not pollute pure domain packages.

---

### [AG007] Field Injection (`@Autowired`)
- **Severity:** `WARNING`
- **Rationale:** Field injection hides dependencies, prevents immutability (`final` fields), complicates unit testing without reflection, and masks circular dependencies.
- ❌ **Violation:**
  ```java
  @Service
  public class PaymentService {
      @Autowired
      private PaymentGateway gateway;
  }
  ```
- ✅ **Remediation:**
  ```java
  @Service
  public class PaymentService {
      private final PaymentGateway gateway;
      public PaymentService(PaymentGateway gateway) {
          this.gateway = gateway;
      }
  }
  ```

---

### [AG008] Swallowed Exception
- **Severity:** `WARNING`
- **Rationale:** Empty `catch` blocks or catching exceptions without logging or rethrowing cause silent failures in production that are nearly impossible to diagnose.
- ❌ **Violation:**
  ```java
  try {
      doCriticalWork();
  } catch (Exception e) {
      // Empty catch block!
  }
  ```
- ✅ **Remediation:**
  ```java
  try {
      doCriticalWork();
  } catch (Exception e) {
      log.error("Failed to process critical work", e);
      throw new BusinessProcessingException("Operation failed", e);
  }
  ```

---

### [AG009] Missing `@Transactional` on Multi-Repository Operations
- **Severity:** `WARNING`
- **Rationale:** Public service methods performing multiple database write/update operations across repositories without a transaction boundary risk partial commits. If the second repository call fails, the first write remains committed in the database, causing silent data corruption and inconsistent state.
- ❌ **Violation (Missing Transaction Boundary):**
  ```java
  @Service
  public class OrderProcessingService {
      private final OrderRepository orderRepository;
      private final AuditRepository auditRepository;

      // Missing @Transactional! If auditRepository fails, order remains saved!
      public void completeOrder(Order order, AuditRecord audit) {
          orderRepository.save(order);
          auditRepository.record(audit);
      }
  }
  ```
- ✅ **Standard Remediation:**
  ```java
  @Service
  public class OrderProcessingService {
      private final OrderRepository orderRepository;
      private final AuditRepository auditRepository;

      @Transactional // Guarantees both calls succeed or both roll back together
      public void completeOrder(Order order, AuditRecord audit) {
          orderRepository.save(order);
          auditRepository.record(audit);
      }
  }
  ```

#### 💡 Architectural Pattern: Handling Remote HTTP / 3rd-Party I/O with Multi-Repo Writes
> **⚠️ Critical HikariCP Performance Warning:** Never place slow remote HTTP calls (e.g. Stripe, REST clients, Feign) **inside** a `@Transactional` method. Doing so locks the database connection for seconds while waiting for network I/O, quickly exhausting the database connection pool.

To satisfy atomic multi-repository writes without locking database connections during remote I/O, use one of these two enterprise patterns:

**Pattern A: Programmatic `TransactionTemplate` (Recommended for Co-located Logic)**
```java
@Service
public class OrderCheckoutService {
    private final OrderRepository orderRepository;
    private final AuditRepository auditRepository;
    private final RestClient paymentClient;
    private final TransactionTemplate transactionTemplate;

    // Method itself is NOT @Transactional (does not hold DB connection during HTTP call)
    public void checkout(OrderRequest request) {
        // 1. Remote HTTP call executed with ZERO DB connections locked
        PaymentResult payment = paymentClient.post().uri("/pay").retrieve().body(PaymentResult.class);

        // 2. Atomic multi-repository write executed inside dedicated transaction boundary
        transactionTemplate.executeWithoutResult(status -> {
            orderRepository.save(new Order(request, payment));
            auditRepository.record(new AuditRecord(request.id(), payment));
        });
    }
}
```

**Pattern B: The Orchestrator Pattern (Recommended for Hexagonal / Clean Architecture)**
Separate orchestration from database persistence:
```java
// 1. Non-transactional Orchestrator: Coordinates network I/O
@Service
public class OrderOrchestrator {
    private final RestClient paymentClient;
    private final OrderPersistenceService persistenceService;

    public void checkout(OrderRequest request) {
        PaymentResult payment = paymentClient.post().uri("/pay").retrieve().body(PaymentResult.class);
        persistenceService.saveOrderAndAudit(request, payment); // Delegates to atomic DB service
    }
}

// 2. Dedicated Transactional Service: Handles atomic repository mutations
@Service
public class OrderPersistenceService {
    private final OrderRepository orderRepository;
    private final AuditRepository auditRepository;

    @Transactional // Satisfies AG009 without holding connections during remote I/O
    public void saveOrderAndAudit(OrderRequest request, PaymentResult payment) {
        orderRepository.save(new Order(request, payment));
        auditRepository.record(new AuditRecord(request.id(), payment));
    }
}
```

---

### [AG010] Unsafe Mutable Static State
- **Severity:** `ERROR`
- **Rationale:** Non-final static fields or mutable static collections (`static List`, `static Map`) in concurrent server environments cause race conditions, cross-request memory leaks, and non-deterministic bugs.
- ❌ **Violation:**
  ```java
  public class UserSessionCache {
      public static List<User> activeUsers = new ArrayList<>(); // Unsafe shared state!
  }
  ```
- ✅ **Remediation:** Use thread-safe Spring beans (e.g., singleton service with `ConcurrentHashMap` or an external cache like Redis).

---

### [AG011] Direct Console Logging
- **Severity:** `WARNING`
- **Rationale:** Direct calls to `System.out.println()`, `System.err.println()`, or `e.printStackTrace()` block I/O threads, cannot be filtered by log level in production, and bypass central log aggregators (e.g., ELK / Datadog).
- ❌ **Violation:**
  ```java
  System.out.println("Processing order: " + orderId);
  e.printStackTrace();
  ```
- ✅ **Remediation:**
  ```java
  private static final Logger log = LoggerFactory.getLogger(OrderService.class);
  // Or Lombok: @Slf4j
  log.info("Processing order: {}", orderId);
  log.error("Order processing failed", e);
  ```

---

### [AG012] Direct Entity Exposure in REST API
- **Severity:** `ERROR`
- **Rationale:** Exposing database JPA `@Entity` classes directly in `@RestController` request/response signatures causes mass-assignment security vulnerabilities, unintended DB modifications, lazy loading serialization exceptions (`LazyInitializationException`), and exposes DB schema details to API consumers.
- ❌ **Violation:**
  ```java
  @GetMapping("/users/{id}")
  public UserEntity getUser(@PathVariable Long id) { // Exposing JPA entity!
      return userService.findById(id);
  }
  ```
- ✅ **Remediation:** Use Java 21 Records or DTOs:
  ```java
  public record UserResponse(Long id, String name, String email) {}

  @GetMapping("/users/{id}")
  public UserResponse getUser(@PathVariable Long id) {
      return userService.getUserDto(id);
  }
  ```

---

### [AG013] Hardcoded Secrets Scanner
- **Severity:** `ERROR`
- **Rationale:** Committing plaintext passwords, database credentials, AWS access keys, or JWT private keys directly into Java source code is a major security risk.
- ❌ **Violation:**
  ```java
  private String apiKey = "AKIAIOSFODNN7EXAMPLE";
  private String dbPassword = "password1234!";
  ```
- ✅ **Remediation:** Load credentials from environment variables or Spring Cloud Vault / AWS Secrets Manager:
  ```java
  @Value("${app.security.api-key}")
  private String apiKey;
  ```

---

### [AG014] JPA Eager Fetch / N+1 Query Risk
- **Severity:** `WARNING` (Advisory)
- **Rationale:** Using `FetchType.EAGER` in JPA relationship mappings causes Cartesian explosion in SQL queries, excessive memory consumption, and N+1 query performance degradation.
- ❌ **Violation:**
  ```java
  @Entity
  public class OrderEntity {
      @OneToMany(fetch = FetchType.EAGER)
      private List<OrderItemEntity> items; // Eager load causes Cartesian product queries!
  }
  ```
- ✅ **Remediation:**
  ```java
  @Entity
  public class OrderEntity {
      @OneToMany(fetch = FetchType.LAZY)
      private List<OrderItemEntity> items;
  }
  ```

---

### [AG015] Missing `@Valid` on REST `@RequestBody`
- **Severity:** `WARNING` (Advisory)
- **Rationale:** Accepting mutation payloads without `@Valid` or `@Validated` bypasses Bean Validation annotations (`@NotNull`, `@Size`, `@Pattern`), allowing unvalidated data to reach domain logic.
- ❌ **Violation:**
  ```java
  @PostMapping("/users")
  public UserResponse createUser(@RequestBody CreateUserRequest request) { // Missing @Valid!
      return userService.create(request);
  }
  ```
- ✅ **Remediation:**
  ```java
  @PostMapping("/users")
  public UserResponse createUser(@Valid @RequestBody CreateUserRequest request) {
      return userService.create(request);
  }
  ```

---

### [AG016] Unpaginated Collection in REST Controller
- **Severity:** `WARNING` (Advisory)
- **Rationale:** Returning raw unpaginated `List<T>` or `Set<T>` from GET endpoints risks OutOfMemoryError and severe database latency as datasets scale.
- ❌ **Violation:**
  ```java
  @GetMapping("/orders")
  public List<OrderResponse> getAllOrders() { // Unpaginated list!
      return orderService.findAll();
  }
  ```
- ✅ **Remediation:**
  ```java
  @GetMapping("/orders")
  public Page<OrderResponse> getAllOrders(Pageable pageable) {
      return orderService.findAll(pageable);
  }
  ```

---

### [AG017] Injected Dependency Bloat / God Class
- **Severity:** `WARNING` (Advisory)
- **Rationale:** Classes with $\ge 7$ injected dependencies indicate a violation of the Single Responsibility Principle, creating tightly coupled God Classes that are difficult to test and maintain.
- ✅ **Remediation:** Refactor into smaller domain services or introduce Facade / Strategy patterns to reduce collaborator count.

---

### [AG018] High Method Cyclomatic & Cognitive Complexity
- **Severity:** `WARNING` (Advisory)
- **Rationale:** Methods with cyclomatic complexity $> 15$ contain excessive conditional branching (`if/else`, `switch`, loops, ternary operators), leading to high cognitive load, elevated defect rates, and difficult unit testing.
- ❌ **Violation:** Monolithic 120-line methods with 5 nested `if/for` branches and complex switch statements.
- ✅ **Remediation:** Extract independent logic into private single-responsibility helper methods or replace branching with a Polymorphic Strategy pattern.

---

### [AG019] Algorithmic Performance Risk ($O(n^2)$ Lookups & DB in Loops)
- **Severity:** `WARNING` (Advisory)
- **Rationale:** Executing linear collection searches (`List.contains()`, `List.indexOf()`) or remote database calls (`repository.findById()`) inside iteration loops causes quadratic $O(n^2)$ CPU bottlenecks or N+1 remote database roundtrip latency.
- ❌ **Violation:**
  ```java
  for (Order order : orderList) {
      if (processedOrderIds.contains(order.getId())) { // O(n*m) scan on List!
          userRepository.findById(order.getUserId()); // N+1 Database queries in loop!
      }
  }
  ```
- ✅ **Remediation:** Convert lookup collections to `HashSet` for $O(1)$ lookups and batch database queries:
  ```java
  Set<Long> processedSet = new HashSet<>(processedOrderIds); // O(1) set
  List<User> users = userRepository.findAllById(userIds); // Single batch query!
  ```

---

### [AG020] Code Readability & Clean Code Antipatterns
- **Severity:** `WARNING` (Advisory)
- **Rationale:** Poorly structured, oversized classes and complex nested methods dramatically increase cognitive load, make peer reviews error-prone, and indicate God Class / Spaghetti Code smells:
  - **Bloated Class/File ($> 450$ lines):** Indicates multiple responsibilities crammed into a single file.
  - **Excessive Method Count ($> 25$ methods):** High public surface area and low cohesion.
  - **Monster Method ($> 60$ lines):** Violates Single Responsibility and does too much.
  - **Long Parameter List ($\ge 5$ parameters):** Parameter explosion and tight coupling.
  - **Deep Nesting / Arrow Antipattern ($\ge 4$ levels):** Deeply nested `if/for/try` blocks impair readability.
- ❌ **Violation:**
  ```java
  public void processBatch(String id, String type, int priority, boolean retry, List<String> tags) {
      if (id != null) {
          for (String tag : tags) {
              if (tag.startsWith("X")) {
                  try {
                      // Level 4+ nesting, 75 lines long!
                  } catch (Exception e) { ... }
              }
          }
      }
  }
  ```
- ✅ **Remediation:**
  1. Use **Guard Clauses (Return Early)**: `if (id == null) return;` to flatten indentation.
  2. Encapsulate long parameter lists into a Java 21 `record RequestParams(...)`.
  3. Extract cohesive sub-blocks into descriptive private helper methods (`extractTagData()`, `validateBatch()`).
  4. Decompose bloated classes into focused domain components.

---

### [AG021] Spring Self-Invocation Proxy Bypass Detection
- **Severity:** `ERROR`
- **Rationale:** In Spring, proxy-driven features (`@Transactional`, `@Async`, `@Cacheable`, `@CacheEvict`, `@Retryable`, `@Scheduled`) work by intercepting calls through an AOP proxy bean. When a method calls another annotated method in the **same class** via `this.method()` or direct call, the call bypasses the Spring proxy entirely, silently executing without transaction management, asynchronous threading, or cache interception!
- ❌ **Violation:**
  ```java
  @Service
  public class OrderProcessingService {

      public void processOrderBatch(List<Order> orders) {
          for (Order order : orders) {
              // Self-invocation: calls @Transactional method internally bypassing Spring proxy!
              saveOrderWithAudit(order);
          }
      }

      @Transactional
      public void saveOrderWithAudit(Order order) {
          orderRepository.save(order);
          auditRepository.record(order);
      }
  }
  ```
- ✅ **Remediation:**
  1. **Extract to a separate Spring Bean:** Move the transactional/async method to a dedicated helper service (e.g. `OrderTransactionService`).
  2. **Inject Self-Proxy / ObjectProvider:** Inject `ObjectProvider<OrderProcessingService>` or self-reference.
  3. **Use TransactionTemplate:** Use programmatic `TransactionTemplate.execute(...)` instead of method-level `@Transactional`.

---

### [AG022] Singleton Spring Bean Mutable Instance State
- **Severity:** `ERROR`
- **Rationale:** Spring beans (`@Service`, `@Component`, `@RestController`, `@Repository`) are singletons by default and shared across concurrent HTTP request threads. Defining non-final mutable fields, mutable collections, or non-thread-safe formatters (`SimpleDateFormat`, `Calendar`, `DecimalFormat`) creates severe multithreaded race conditions, corrupted user session state, and cross-request data leaks.
- ❌ **Violation:**
  ```java
  @Service
  public class OrderProcessingService {
      private List<Order> currentBatch = new ArrayList<>(); // Shared state across all HTTP requests!
      private SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd"); // Not thread-safe!
      private int processedCount = 0; // Multithreaded race condition!
  }
  ```
- ✅ **Remediation:**
  1. Make fields `final` and immutable (or inject stateless Spring beans).
  2. Scope state to method execution (thread-local on call stack).
  3. Replace `SimpleDateFormat` with thread-safe `java.time.format.DateTimeFormatter`.

---

### [AG023] NullPointer Traps & Defensive Coding Antipatterns
- **Severity:** `WARNING` (Advisory)
- **Rationale:** Detects high-probability NullPointer bugs and defensive coding anti-patterns:
  - **Raw `Optional.get()` without check:** Calling `findUser().get()` without `.isPresent()` throws `NoSuchElementException`.
  - **Returning `null` Collection / Optional:** Returning `return null;` from a method with return type `List<T>`, `Set<T>`, `Map<K,V>`, or `Optional<T>` forces callers to do manual null checks or crash.
  - **Inverted Literal Equals:** `variable.equals("ADMIN")` throws `NullPointerException` if `variable` is null.
- ❌ **Violation:**
  ```java
  public List<Order> getOrders(String status) {
      if (status.equals("COMPLETED")) { // Throws NPE if status is null!
          User user = userRepository.findById(1L).get(); // Throws NoSuchElementException if absent!
          return null; // Forces callers to null-check collections!
      }
      return orderService.findAll();
  }
  ```
- ✅ **Remediation:**
  ```java
  public List<Order> getOrders(String status) {
      if ("COMPLETED".equals(status)) { // Null-safe Yoda equality
          User user = userRepository.findById(1L).orElseThrow(() -> new UserNotFoundException(1L));
          return Collections.emptyList(); // Safe empty collection
      }
      return orderService.findAll();
  }
  ```

---

### [AG024] Unclosed AutoCloseable Resource / I/O Stream Leaks
- **Severity:** `ERROR`
- **Rationale:** Instantiating `AutoCloseable` I/O streams, readers, or database resources (`FileInputStream`, `FileOutputStream`, `BufferedReader`, `Connection`, `PreparedStatement`, `ResultSet`, `Scanner`) outside of a **try-with-resources** statement leads to leaked operating system file descriptors and exhausted database connection pools.
- ❌ **Violation:**
  ```java
  public void exportData(File file) throws IOException {
      FileInputStream fis = new FileInputStream(file); // Leaks file handle if exception occurs!
      byte[] data = fis.readAllBytes();
      fis.close();
  }
  ```
- ✅ **Remediation:**
  ```java
  public void exportData(File file) throws IOException {
      try (FileInputStream fis = new FileInputStream(file)) { // Guaranteed automatic cleanup!
          byte[] data = fis.readAllBytes();
      }
  }
  ```

---

### [AG025] Inefficient String Concatenation in Loops
- **Severity:** `WARNING` (Advisory)
- **Rationale:** Repeated string concatenation using `+` or `+=` inside loops (`for`, `while`, `do-while`) creates intermediate `String` objects on every iteration, leading to $O(n^2)$ heap allocation and severe Garbage Collection pauses.
- ❌ **Violation:**
  ```java
  String csv = "";
  for (String row : rows) {
      csv += row + "\n"; // Allocates new String copy on every iteration!
  }
  ```
- ✅ **Remediation:**
  ```java
  StringBuilder csv = new StringBuilder();
  for (String row : rows) {
      csv.append(row).append("\n");
  }
  ```

---

### [AG026] Remote I/O in `@Transactional` Boundary
- **Severity:** `ERROR`
- **Rationale:** Executing remote HTTP, REST, RPC, email, or cloud SDK calls inside a `@Transactional` boundary locks the database connection from HikariCP pool idle for seconds while waiting for network responses. Under concurrent load, connection pool starvation occurs in milliseconds, crashing the entire microservice (`Connection is not available, request timed out after 30000ms`).
- ❌ **Violation:**
  ```java
  @Service
  public class OrderCheckoutService {
      @Autowired private OrderRepository orderRepository;
      @Autowired private RestClient paymentClient;

      @Transactional // Holds physical DB connection locked during external network call!
      public void checkout(OrderRequest request) {
          Order order = orderRepository.save(new Order(request));

          // Remote HTTP call takes 2-5 seconds! HikariCP connection held idle!
          PaymentResponse payment = paymentClient.post()
                  .uri("/charge")
                  .body(request)
                  .retrieve()
                  .body(PaymentResponse.class);

          order.setStatus(payment.status());
          orderRepository.save(order);
      }
  }
  ```
- ✅ **Remediation:**
  1. **Use Programmatic `TransactionTemplate`:** Keep the remote HTTP call outside the transaction boundary and wrap only the database write operations inside `transactionTemplate.executeWithoutResult(...)`.
  2. **Use the Orchestrator Pattern:** Separate concerns into a non-transactional orchestrator service (which coordinates HTTP calls) and a dedicated `@Transactional` persistence service.

---

### [AG027] Java 21 Virtual Thread Pinning
- **Severity:** `WARNING`
- **Rationale:** When running on Java 21 Virtual Threads (Project Loom), entering a `synchronized` method or `synchronized(lock)` block **pins** the virtual thread to its underlying OS carrier thread. If blocking I/O (database, socket, REST call) occurs inside the synchronized block, the carrier thread is frozen, starving the virtual thread scheduler and dropping concurrency throughput from 50,000 req/sec down to 10 req/sec.
- ❌ **Violation:**
  ```java
  @Service
  public class CurrencyExchangeService {
      private final Map<String, BigDecimal> cache = new HashMap<>();

      // Synchronized method pins OS carrier thread in Java 21!
      public synchronized BigDecimal getExchangeRate(String currencyPair) {
          if (!cache.containsKey(currencyPair)) {
              cache.put(currencyPair, fetchRateFromRemoteOrDb(currencyPair)); // Blocking I/O while pinned!
          }
          return cache.get(currencyPair);
      }
  }
  ```
- ✅ **Remediation:**
  1. Replace `synchronized` with `java.util.concurrent.locks.ReentrantLock`, which properly unmounts virtual threads without pinning carrier threads:
     ```java
     private final ReentrantLock lock = new ReentrantLock();

     public BigDecimal getExchangeRate(String currencyPair) {
         lock.lock(); // Safe: does NOT pin carrier threads
         try {
             return cache.computeIfAbsent(currencyPair, this::fetchRateFromRemoteOrDb);
         } finally {
             lock.unlock();
         }
     }
     ```
  2. Use concurrent non-blocking data structures like `ConcurrentHashMap.computeIfAbsent(...)`.

---

### [AG028] ThreadLocal Memory Leak & Context Contamination
- **Severity:** `ERROR`
- **Rationale:** Web servlet containers (Tomcat, Jetty, Undertow) use fixed worker thread pools to handle incoming HTTP requests. Calling `ThreadLocal.set()` without calling `ThreadLocal.remove()` in an enclosing `finally` block leaves values attached to the pooled `Thread` instance indefinitely. This causes:
  1. **Cross-Tenant Data Contamination / Security Breaches:** A subsequent request from User B served by the same recycled worker thread reads User A's tenant ID, user token, or security permissions.
  2. **Metaspace & Heap ClassLoader Leaks:** If custom classes or classloaders are retained in `ThreadLocal` references during dynamic application redeployment, the GC cannot collect the classloader, causing `OutOfMemoryError: Metaspace`.
- ❌ **Violation:**
  ```java
  @Service
  public class UserContextService {
      private static final ThreadLocal<SecurityContext> contextHolder = new ThreadLocal<>();

      public void processRequest(SecurityContext context) {
          contextHolder.set(context); // Leaks into thread pool if an exception is thrown or method exits!
          businessLogic();
          // Even if contextHolder.remove() is here, an uncaught exception skips it!
      }
  }
  ```
- ✅ **Remediation:**
  1. Always wrap usage in a `try-finally` block ensuring `.remove()` is executed under all exit paths:
     ```java
     contextHolder.set(context);
     try {
         businessLogic();
     } finally {
         contextHolder.remove(); // Guaranteed cleanup upon return or exception
     }
     ```
  2. In Java 21+, migrate to immutable **`ScopedValue`** (JEP 446 / JEP 429), which automatically bounds value lifetime to the syntactic execution scope without thread leakage risk.

---

### [AG029] Unbounded `@Async` Thread Pool Exhaustion
- **Severity:** `WARNING`
- **Rationale:** In Spring Framework and Spring Boot, applying `@Async` without specifying a bean qualifier (e.g. `@Async("taskExecutor")`) causes Spring to search for an `Executor` bean. If no single unique `TaskExecutor` bean is present or configured, Spring defaults to `SimpleAsyncTaskExecutor`. 
  `SimpleAsyncTaskExecutor` **does not reuse threads**; it spawns a brand-new OS platform thread for every single asynchronous invocation. Under moderate-to-high traffic bursts, thousands of concurrent threads are spawned, resulting in immediate native thread exhaustion (`java.lang.OutOfMemoryError: unable to create new native thread`) and JVM crash.
- ❌ **Violation:**
  ```java
  @Service
  public class NotificationService {

      @Async // Missing executor qualifier! Falls back to unbounded SimpleAsyncTaskExecutor!
      public void sendEmailNotification(String orderId) {
          emailGateway.send(orderId);
      }
  }
  ```
- ✅ **Remediation:**
  1. Define a bounded, production-ready `ThreadPoolTaskExecutor` bean with bounded queue and rejection policy in your configuration:
     ```java
     @Configuration
     @EnableAsync
     public class AsyncConfig {
         @Bean("boundedTaskExecutor")
         public Executor boundedTaskExecutor() {
             ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
             executor.setCorePoolSize(10);
             executor.setMaxPoolSize(50);
             executor.setQueueCapacity(500);
             executor.setThreadNamePrefix("async-worker-");
             executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
             executor.initialize();
             return executor;
         }
     }
     ```
  2. Explicitly specify the qualified bean name in `@Async`:
     ```java
     @Async("boundedTaskExecutor") // Bounded execution prevents thread starvation
     public void sendEmailNotification(String orderId) {
         emailGateway.send(orderId);
     }
     ```

---

### [AG030] Missing HTTP Client Timeout Configuration
- **Severity:** `ERROR`
- **Rationale:** In Spring Framework and standard Java, default HTTP client implementations (`RestTemplate`, `RestClient`, and `HttpClient`) are initialized with **infinite (0ms) default timeouts**. If a downstream third-party service, payment processor, or microservice degrades or hangs, calling worker threads block indefinitely. Under production traffic, all Tomcat/Jetty worker threads freeze within seconds, leading to connection pool exhaustion, unresponsiveness, and cascading cluster failure.
- ❌ **Violation:**
  ```java
  // 1. Un-timed raw RestTemplate returned as a Spring @Bean
  @Configuration
  public class NetworkConfig {
      @Bean
      public RestTemplate restTemplate() {
          return new RestTemplate(); // Default timeout is INFINITE!
      }
  }

  // 2. Direct creation of un-timed RestTemplate inside Spring Service
  @Service
  public class PaymentService {
      public void pay() {
          RestTemplate rt = new RestTemplate(); // Infinite timeout and unpooled!
          rt.postForEntity(...);
      }
  }

  // 3. Immutable standard Java HttpClient created without connect timeout
  HttpClient client = HttpClient.newHttpClient(); // Connect timeout is empty!
  ```
- ✅ **Remediation:**
  1. For `RestTemplate`, configure timeouts via Spring Boot's `RestTemplateBuilder`:
     ```java
     @Bean
     public RestTemplate restTemplate(RestTemplateBuilder builder) {
         return builder
             .setConnectTimeout(Duration.ofSeconds(3))
             .setReadTimeout(Duration.ofSeconds(5))
             .build();
     }
     ```
  2. For `RestClient` (Spring Boot 3.2+), pass a configured request factory:
     ```java
     @Bean
     public RestClient restClient(RestClient.Builder builder) {
         SimpleClientHttpRequestFactory factory = new SimpleClientHttpRequestFactory();
         factory.setConnectTimeout(3000);
         factory.setReadTimeout(5000);
         return builder.requestFactory(factory).build();
     }
     ```
  3. For standard Java `HttpClient`, use the builder pattern:
     ```java
     HttpClient client = HttpClient.newBuilder()
         .connectTimeout(Duration.ofSeconds(5))
         .build();
     ```

### [AG031] Dynamic Query Concatenation Injection Risk
- **Severity:** `ERROR`
- **Standard Mapping:** OWASP Top 10 (A03:2021 — Injection), CWE-89
- **Rationale:** Concatenating dynamic user input or runtime variables directly into SQL or JPQL query strings bypasses query plan caching and exposes systems to critical SQL/JPQL injection attacks.
- ❌ **Violation:**
  ```java
  @Query("SELECT u FROM User u WHERE u.username = '" + input + "'")
  User findUser(String input);

  em.createQuery("SELECT o FROM Order o WHERE o.id = " + orderId);
  ```
- ✅ **Remediation:**
  ```java
  @Query("SELECT u FROM User u WHERE u.username = :username")
  User findUser(@Param("username") String username);

  em.createQuery("SELECT o FROM Order o WHERE o.id = :orderId", Order.class)
    .setParameter("orderId", orderId);
  ```

### [AG032] Broken Cryptographic Algorithm
- **Severity:** `ERROR`
- **Standard Mapping:** OWASP Top 10 (A02:2021 — Cryptographic Failures), CWE-327
- **Rationale:** Obsolete hash algorithms (`MD5`, `SHA-1`) suffer from collision and preimage vulnerabilities. Weak ciphers (`DES`, `RC4`) and ECB mode leak plaintext block patterns and can be decrypted in minutes.
- ❌ **Violation:**
  ```java
  MessageDigest md = MessageDigest.getInstance("MD5");
  Cipher cipher = Cipher.getInstance("AES/ECB/PKCS5Padding");
  ```
- ✅ **Remediation:**
  ```java
  MessageDigest md = MessageDigest.getInstance("SHA-256");
  Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
  ```

### [AG033] Insecure CORS Configuration
- **Severity:** `ERROR`
- **Standard Mapping:** OWASP Top 10 (A05:2021 — Security Misconfiguration), CWE-942
- **Rationale:** Allowing wildcard origins (`*`) while enabling credentials (`allowCredentials = true`) allows any arbitrary external website to perform authenticated requests on behalf of logged-in users, leaking confidential data or session tokens.
- ❌ **Violation:**
  ```java
  @CrossOrigin(origins = "*", allowCredentials = "true")
  @RestController
  public class AccountController { ... }
  ```
- ✅ **Remediation:**
  ```java
  @CrossOrigin(origins = "https://app.example.com", allowCredentials = "true")
  @RestController
  public class AccountController { ... }
  ```

---

## 8. In-Code Rule Suppressions

ArchGuard supports fine-grained, auditable in-code suppressions on **classes** and **methods** when intentional architectural exceptions must be documented in code.

### Auditable Quality Gate Semantics
- **Quality Gate (CI/CD)**: In-code suppressions **do not fail the build** (`mvn verify` succeeds) and do not degrade the module health score.
- **Architectural Auditability**: Suppressed violations are **not hidden or silently swallowed**. They remain fully visible in `architecture-guard-report.html` under the dedicated **`🛡️ Suppressed (N)`** scope tab, marked with **`🛡️ SUPPRESSED IN CODE`** badges, allowing architects and lead reviewers to audit intentional technical debt easily.

---

### 1. Standard Java `@SuppressWarnings` *(Recommended)*

No external dependencies or imports needed. ArchGuard parses standard `@SuppressWarnings` annotations directly via AST source inspection.

#### Method-Level Suppression (Best Practice)
Target specific methods to avoid blinding other methods in the same class:

```java
// Suppress a single rule on a method
@SuppressWarnings("archguard:AG020")
public void complexLegacyMethod() { ... }

// Suppress multiple specific rules on a method
@SuppressWarnings({"archguard:AG018", "archguard:AG019", "archguard:AG020"})
public void processOrderBatch(...) { ... }

// Blanket suppression: suppress ALL ArchGuard rules on this method
@SuppressWarnings("archguard")
public void legacyBatchImport() { ... }
```

#### Class-Level Suppression
Suppresses matching rules across the entire class:

```java
// Suppress a specific rule across the class
@SuppressWarnings("archguard:AG001")
public class LegacyOrderController { ... }

// Suppress multiple rules across the class
@SuppressWarnings({"archguard:AG007", "archguard:AG011"})
public class InternalDiagnosticService { ... }

// Blanket suppression: suppress ALL ArchGuard rules for this class
@SuppressWarnings("archguard")
public class ThirdPartyGeneratedBridge { ... }
```

---

### 2. Dedicated `@SuppressArchitecture` Annotation

When using `architecture-guard-core` directly, you can document the architectural rationale explicitly using `@SuppressArchitecture`:

```java
import com.architectureguard.core.annotation.SuppressArchitecture;

// Suppress specific rule with mandatory audit rationale
@SuppressArchitecture(rule = "AG001", reason = "Legacy adapter requires direct repo access until v2 migration")
public class LegacyOrderController {
    // ...
}

// Suppress multiple rules on class or method
@SuppressArchitecture(rules = {"AG007", "AG011"}, reason = "CLI debugging tool requires field injection and console output")
public class InternalDiagnosticService {
    // ...
}

// Suppress all rules
@SuppressArchitecture("ALL")
public class ThirdPartyGeneratedBridge {
    // ...
}
```

---

## 9. Configuration File Reference (`archguard.yml`)

ArchGuard embraces a **Zero-Configuration** philosophy: if no configuration file exists, it runs with sensible defaults (all 33 rules active, Spring/JPA stereotype layer detection, and default reporting).

When your team needs to **customize** architectural rules, tweak thresholds, map non-standard layer packages, or enforce strict package boundaries, you can add an `archguard.yml` file to your project root.

---

### How to Generate `archguard.yml`
You can create the file manually or scaffold it using the interactive initialization goal/task:

**Maven:**
```bash
mvn architecture-guard:init
```

**Gradle:**
```bash
./gradlew archGuardInit
```

**What this command does:**
1. Scans `src/main/java` to auto-detect your project's root package (e.g. `com.mycompany.service`).
2. Generates a production-ready, commented `archguard.yml` in the project root directory (alongside `pom.xml` or `build.gradle`).
3. Automatically takes effect on all subsequent `mvn clean verify` or `./gradlew check` builds.

---

### File Discovery & Precedence
ArchGuard automatically searches for configuration files using the following precedence ladder:
1. `archguard.yml`
2. `archguard.yaml`
3. `.archguard.yml`
4. `archguard.json`

#### Multi-Module Upward Discovery
In Maven multi-module projects, submodules do not need duplicate configuration files. If a submodule does not have its own `archguard.yml`, ArchGuard **automatically walks up the directory hierarchy (up to 4 parent levels)** to discover the root configuration file. This allows single-point governance across the entire repository.

*(You can also explicitly specify a custom path in `pom.xml` using `<configFile>path/to/my-config.yml</configFile>` or via CLI `-Darchguard.configFile=...`)*.

---

### Complete `archguard.yml` Schema Reference

```yaml
# ==============================================================================
# Architecture Guard Configuration
# Documentation: https://github.com/javatechgroup/archGaurdDocs
# ==============================================================================

# 1. Global Analysis Settings (overrides plugin defaults if set)
analysis:
  failOnSeverity: ERROR             # Build failure threshold: ERROR, WARNING, INFO, NONE
  failOnNewViolations: true         # When baseline is present, only fail on new issues
  threads: 4                        # Worker threads: 0 or omitted = auto-detect all cores, 1 = sequential debug
  outputDir: target/architecture-guard # Target directory for generated reports
  baselineFile: .architecture-guard-baseline.json # Snapshot file for legacy technical debt

# 2. Layer Package Mapping
# Map custom package structures to architectural layers (AG001 - AG004).
# If omitted, ArchGuard automatically detects layers via Spring annotations
# (@RestController, @Service, @Repository, @Entity) and package conventions (*.controller.*, etc.).
packages:
  controller:
    - com.mycompany.app.controller.*
    - com.mycompany.app.web.*
    - com.mycompany.app.api.*
  service:
    - com.mycompany.app.service.*
    - com.mycompany.app.core.*
  repository:
    - com.mycompany.app.repository.*
    - com.mycompany.app.dao.*
  entity:
    - com.mycompany.app.model.*
    - com.mycompany.app.entity.*

# 3. Rule Overrides & Toggles
# Enable/disable specific rules or override their default severity.
rules:
  AG001:
    enabled: true
    severity: ERROR                 # Controller -> Repository bypass
  AG015:
    enabled: true
    severity: WARNING               # Missing @Valid on @RequestBody (downgrade or disable)
  AG021:
    enabled: true
    severity: ERROR                 # Spring AOP proxy self-invocation
  AG022:
    enabled: true
    severity: ERROR                 # Singleton mutable state / concurrency hazards
  AG026:
    enabled: true
    severity: ERROR                 # Remote I/O in @Transactional
  AG027:
    enabled: true
    severity: WARNING               # Java 21 Virtual Thread carrier pinning

# 4. Strict Package Boundary Rules
# Define custom directional access rules between specific packages:
# "key package may ONLY import from the listed packages"
packageBoundaries:
  com.mycompany.app.controller:
    - com.mycompany.app.service
    - com.mycompany.app.dto
  com.mycompany.app.repository:
    - com.mycompany.app.model

# 5. Global Code Exclusions (Ant-style glob patterns)
# Completely excluded from all architectural and code quality inspection
exclusions:
  - "**/generated/**"               # MapStruct, OpenAPI, or Lombok generated code
  - "**/*Test.java"                 # Test sources (analyzed separately if configured)
  - "com.mycompany.app.dto.*"       # Pure transfer objects

# 6. Report Output Formats
reportFormats:
  - CONSOLE                         # Colorized terminal summary
  - HTML                            # Self-contained interactive dark/light dashboard
  - SARIF                           # Standard OASIS SARIF 2.1.0 for GitHub / GitLab PR annotations
  - JSON                            # Machine-readable report for automation
```

---

### Custom Architecture Example: Hexagonal / Clean Architecture

If your service follows Hexagonal (Ports & Adapters) or Clean Architecture instead of classical 3-tier MVC, map your ports and adapters directly:

```yaml
packages:
  controller:
    - com.mycompany.order.adapter.in.web.*
    - com.mycompany.order.adapter.in.rest.*
  service:
    - com.mycompany.order.application.service.*
    - com.mycompany.order.port.in.*
  repository:
    - com.mycompany.order.adapter.out.persistence.*
    - com.mycompany.order.port.out.*
  entity:
    - com.mycompany.order.domain.model.*

packageBoundaries:
  com.mycompany.order.domain:
    # Domain core must be completely isolated and have zero outbound dependencies
    - java.lang
    - java.util
```

---

## 10. Reports, Diagnostics & Automated PR Code Review

> [!TIP]
> **Complete Visual Glossary & Badge Guide:**  
> For an in-depth reference explaining every badge, severity level (`ERROR`, `WARNING`), baseline state (`NEW`, `BASELINED`, `RESOLVED`), and risk indicator (`💥 High Blast Radius`), see the [Report Indicators, Badges & Glossary Guide](file:///e:/Arch-Gaurd/docs/guides/report-glossary.md).

ArchGuard automatically generates comprehensive reporting artifacts in `target/architecture-guard/` (Maven) or `build/reports/architecture-guard/` (Gradle):

1. **`report.sarif` (Standard OASIS SARIF 2.1.0):**
   - Industry-standard Static Analysis Results Interchange Format.
   - Natively supported by **GitHub Code Scanning**, **GitLab Code Quality**, and **Azure DevOps PR annotations** to highlight violations directly on pull request lines of code.

2. **`code-review.md` (Automated Pull Request Code Review):**
   - **Focused on Delta:** Evaluates **only the newly added violations** against `.architecture-guard-baseline.json`. Legacy issues are filtered out.
   - **Automated Verdict:** Generates clear verdict headers (`APPROVED`, `APPROVED WITH COMMENTS`, or `CHANGES REQUESTED`).
   - **File-by-File Review Comments:** Provides exact line numbers, severity emojis (🔴/🟡), issue explanations, and copy-pasteable **Before vs. After remediation snippets**.
   - **Reviewer Checklist:** Auto-generates a clean markdown checklist for peer reviewers and tech leads.

3. **`report.html` (Offline Interactive Dashboard):**
   - **Self-Contained & Offline:** 100% standalone single-file HTML report with embedded CSS/JS and zero external CDN dependencies.
   - **PR Code Review Banner:** Shows review verdict and 1-click filter buttons (`Focus on PR Delta`, `Show Errors Only`).
   - **Sidebar Rule Filtering:** Toggle between **`⚠️ Issues Only`** (shows only rules that have active violations to accelerate review) and **`All Rules`** (full 33-rule catalog), with real-time rule search.
   - **Dedicated `🛡️ Suppressed (N)` Scope:** Auditable tab displaying all code-suppressed violations with distinct **`🛡️ SUPPRESSED IN CODE`** badges without polluting the active failure counts.
   - **One-Click IDE Deep Links:** Normalized links (`vscode://file/...` and `idea://open?file=...`) allowing instant 1-click jumps directly to offending lines in VS Code or IntelliJ IDEA on Windows, macOS, and Linux.
   - **Zero-Results Empty State:** Friendly search and filter feedback with an interactive **`↺ Reset All Filters`** button.
   - **Interactive Code Preview & PR Comment Export:** Click **`🔍 View Source Code`** to preview the exact method snippet or **`📋 Copy PR Comment`** for pre-formatted GitHub/GitLab review comments.

4. **`report.json` (Machine-Readable Output):**
   - Full structured schema containing execution metadata, rule results, SHA-256 fingerprints, and differential breakdowns.

5. **Console Output:**
   - Colorized ANSI console summary during Maven or Gradle builds.

---

## 11. CI/CD Pipeline & Merge Gate Integration

Architecture Guard is engineered from the ground up for automated PR/MR gatekeeping in **GitHub Actions** and **GitLab CI/CD**. Because it generates standard **SARIF 2.1.0** (`report.sarif`), markdown reviews (`code-review.md` and `summary.md`), and differential exit codes, it seamlessly integrates into branch protection policies to block bad merges.

---

### 11.1 GitHub Actions PR Merge Gate

Create a workflow file in your repository at `.github/workflows/architecture-guard.yml`:

```yaml
name: Architecture Quality Gate

on:
  pull_request:
    branches: [ main, master, develop ]
  push:
    branches: [ main, master ]

permissions:
  contents: read
  pull-requests: write     # Required to post automated PR review comments
  security-events: write   # Required to upload SARIF to GitHub Code Scanning

jobs:
  architecture-guard:
    name: Architecture & Quality Verification
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '21'
          cache: 'maven'

      # 1. Execute verification. Fails build if new violations exceed threshold.
      - name: Run Architecture Guard
        run: |
          mvn clean verify \
            -Darchguard.failOnSeverity=ERROR \
            -Darchguard.failOnNewViolationsOnly=true

      # 2. Upload SARIF for inline annotations on PR diffs ("Files changed" tab)
      # Uses glob pattern '**/report.sarif' to support custom outputDirectory & multi-module repos
      - name: Upload SARIF to GitHub Code Scanning
        if: always()
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: '**/report.sarif'
          category: architecture-guard
        continue-on-error: true

      # 3. Post automated Pull Request Code Review comment
      # Dynamically discovers review markdown regardless of custom output directory or submodule location
      - name: Post PR Review Comment
        if: always() && github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const path = require('path');

            function findFile(dir, fileName) {
              const entries = fs.readdirSync(dir, { withFileTypes: true });
              for (const entry of entries) {
                const fullPath = path.join(dir, entry.name);
                if (entry.isDirectory() && entry.name !== '.git' && entry.name !== 'node_modules') {
                  const found = findFile(fullPath, fileName);
                  if (found) return found;
                } else if (entry.isFile() && entry.name === fileName) {
                  return fullPath;
                }
              }
              return null;
            }

            const reviewFile = findFile('.', 'code-review.md') || findFile('.', 'summary.md');
            if (reviewFile) {
              const commentBody = fs.readFileSync(reviewFile, 'utf8');
              github.rest.issues.createComment({
                issue_number: context.issue.number,
                owner: context.repo.owner,
                repo: context.repo.repo,
                body: commentBody
              });
            }

      # 4. Save HTML Dashboard and diagnostic files as workflow artifacts
      # Captures reports across all submodules and custom output folders
      - name: Archive Architecture Reports
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: architecture-guard-reports
          path: |
            **/architecture-guard-report.html
            **/architecture-guard-report.json
            **/code-review.md
            **/report.sarif
          retention-days: 14
```

> [!TIP]
> **Custom Output Directories & Multi-Module Projects:**
> Notice the use of recursive glob patterns (`**/report.sarif`, `**/architecture-guard-report.html`). Whether you configure `<outputDirectory>${project.build.directory}/my-custom-path</outputDirectory>` in `pom.xml`, set `outputDir` in `archguard.yml`, or have multiple submodules (`core/`, `web/`, `service/`), the CI job dynamically locates all generated reports with zero configuration changes!

#### GitHub Branch Protection (Enforce Before Merging)
1. Go to repository **Settings** -> **Branches** -> **Branch protection rules**.
2. Edit rule for `main` or `master`.
3. Check **"Require status checks to pass before merging"**.
4. Search for and select the **`Architecture & Quality Verification`** check.
5. Check **"Require branches to be up to date before merging"**.

---

### 11.2 GitLab CI/CD Merge Request Gate

Add the following job to your `.gitlab-ci.yml`:

```yaml
stages:
  - build
  - test
  - quality-gate

variables:
  MAVEN_OPTS: "-Dmaven.repo.local=.m2/repository"

architecture-guard:
  stage: quality-gate
  image: maven:3.9.6-eclipse-temurin-21
  rules:
    # Run on all Merge Requests and on default branch merges
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
  cache:
    key: "${CI_JOB_NAME}"
    paths:
      - .m2/repository
  script:
    - mvn clean verify -Darchguard.failOnSeverity=ERROR -Darchguard.failOnNewViolationsOnly=true
  artifacts:
    when: always
    name: "architecture-guard-$CI_COMMIT_REF_SLUG"
    paths:
      # Wildcards capture artifacts across custom output paths and all submodules
      - "**/architecture-guard-report.html"
      - "**/architecture-guard-report.json"
      - "**/code-review.md"
      - "**/report.sarif"
    reports:
      # Ingests SARIF directly into GitLab Vulnerability & Code Quality reports
      sast: "**/report.sarif"
    expire_in: 14 days
```

#### Optional: Post Automated MR Note in GitLab
If you want GitLab to post the markdown review comment directly on the Merge Request, add an `after_script` step using GitLab's API (using `find` to dynamically locate the review markdown across any custom output path):

```yaml
  after_script:
    - |
      REVIEW_FILE=$(find . -name "code-review.md" -o -name "summary.md" | head -n 1)
      if [ -n "$REVIEW_FILE" ] && [ -n "$CI_MERGE_REQUEST_IID" ]; then
        curl --request POST \
          --header "PRIVATE-TOKEN: ${GITLAB_COMMENT_TOKEN}" \
          --header "Content-Type: application/json" \
          --data "{\"body\": $(jq -Rs . < "$REVIEW_FILE")}" \
          "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/merge_requests/${CI_MERGE_REQUEST_IID}/notes"
      fi
```

#### GitLab Merge Request Approval Settings
1. Navigate to **Settings** -> **Merge requests**.
2. Under **Merge checks**, enable **"Pipelines must succeed"**.
3. Enable **"All threads must be resolved"**.

---

### 11.3 Brownfield Adoption & Ratchet Strategy (Zero Friction)

For existing codebases with hundreds of legacy violations, adopt this 3-step strategy to prevent disruption while stopping new technical debt immediately:

1. **Step 1: Freeze Current Debt (One-time setup)**
   Run the baseline goal locally and commit the generated file:
   ```bash
   mvn architecture-guard:baseline
   git add .architecture-guard-baseline.json
   git commit -m "chore: snapshot architecture guard baseline"
   git push
   ```
2. **Step 2: Gate Incoming Pull Requests**
   Leave `-Darchguard.failOnNewViolationsOnly=true` enabled in CI.
   - ✅ **Existing legacy violations** are tolerated and will never break a colleague's PR.
   - ❌ **Any new violation** introduced in the PR immediately fails the merge gate.
3. **Step 3: Enable Ratchet Mode on Merge to Default Branch**
   In your CI pipeline for pushes to `main`/`master`, add `-Darchguard.ratchet=true`. As developers refactor legacy code, resolved violations are automatically purged from `.architecture-guard-baseline.json`, locking in architectural improvements forever:
   ```bash
   mvn clean verify -Darchguard.ratchet=true
   ```

---

### 11.4 Rendering & Hosting the HTML Dashboard in CI/CD

Because `architecture-guard-report.html` is **100% self-contained and offline** (all CSS, JavaScript, icons, dark/light theme switcher, and graph engines are bundled into one file with **zero external CDN dependencies**), it can be rendered and hosted anywhere without needing a separate backend server or database.

#### Custom Output Directory in `pom.xml`
By default, reports are saved to `target/architecture-guard/`. You can customize this location in your `pom.xml`, via CLI, or in `archguard.yml`:

```xml
<!-- In your pom.xml -->
<plugin>
    <groupId>io.github.javatechgroup</groupId>
    <artifactId>architecture-guard-maven-plugin</artifactId>
    <version>1.0.5</version>
    <configuration>
        <!-- Custom report destination directory -->
        <outputDirectory>${project.build.directory}/custom-architecture-reports</outputDirectory>
    </configuration>
</plugin>
```

> [!TIP]
> **Resilient CI/CD Pipelines:**
> Always use recursive wildcards (`**/architecture-guard-report.html`, `**/report.sarif`) in your CI steps instead of hardcoding `target/architecture-guard/`. This ensures your CI scripts and upload steps **never fail**, regardless of what custom `<outputDirectory>` individual developers configure in `pom.xml`.

---

#### Approach A: Copying to a Running Web Server or S3 (Recommended for Enterprise)

Both **GitHub Actions** and **GitLab CI** can copy the generated HTML dashboard to an internal company web server (Nginx, Apache, or AWS S3 / CloudFront / MinIO) and comment a **live 1-click link** directly on the Pull Request / Merge Request.

##### 1. Internal Nginx / Apache Server via SSH/Rsync:

* **In GitHub Actions (`.github/workflows/architecture-guard.yml`):**
  ```yaml
        # Dynamically discovers report even with custom outputDirectory
        - name: Deploy Report to Internal Web Server
          if: always()
          run: |
            REPORT_DIR=$(find . -type d -name "*architecture-guard*" -o -name "*custom-architecture-reports*" | head -n 1)
            rsync -avz ${REPORT_DIR}/ deployer@internal-docs.mycompany.com:/var/www/html/reports/${{ github.event.repository.name }}/

        - name: Post Live Dashboard Link to PR
          if: always() && github.event_name == 'pull_request'
          uses: actions/github-script@v7
          with:
            script: |
              const url = `https://internal-docs.mycompany.com/reports/${context.repo.repo}/architecture-guard-report.html`;
              github.rest.issues.createComment({
                issue_number: context.issue.number,
                owner: context.repo.owner,
                repo: context.repo.repo,
                body: `🌐 **Live Architecture Dashboard:** [Click to open interactive report](${url})`
              });
  ```

* **In GitLab CI/CD (`.gitlab-ci.yml`):**
  ```yaml
  architecture-guard:
    stage: quality-gate
    script:
      - mvn clean verify -Darchguard.failOnNewViolationsOnly=true
      - |
        REPORT_DIR=$(find . -name "architecture-guard-report.html" -exec dirname {} \; | head -n 1)
        rsync -avz "${REPORT_DIR}/" deployer@internal-docs.mycompany.com:/var/www/html/reports/${CI_PROJECT_NAME}/
    after_script:
      - |
        if [ -n "$CI_MERGE_REQUEST_IID" ]; then
          URL="https://internal-docs.mycompany.com/reports/${CI_PROJECT_NAME}/architecture-guard-report.html"
          curl --request POST \
            --header "PRIVATE-TOKEN: ${GITLAB_COMMENT_TOKEN}" \
            --header "Content-Type: application/json" \
            --data "{\"body\": \"🌐 **Live Architecture Dashboard:** [Click to open interactive report](${URL})\"}" \
            "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/merge_requests/${CI_MERGE_REQUEST_IID}/notes"
        fi
  ```

##### 2. Cloud Storage (AWS S3 / CloudFront / GCP Cloud Storage):
```bash
# Upload to S3 static website hosting bucket
aws s3 cp $(find . -name "architecture-guard-report.html" | head -n 1) \
  s3://my-company-arch-reports/${PROJECT_NAME}/index.html
```

---

#### Approach B: GitLab Native Direct Browser Viewing (`expose_as`)

GitLab allows reviewers to view the HTML report directly inside the browser from the Merge Request widget without setting up any web servers:

```yaml
  artifacts:
    when: always
    expose_as: "Architecture Guard Dashboard"  # Adds a direct 1-click button in GitLab MR UI!
    paths:
      - "**/architecture-guard-report.html"
```
When reviewers click **`Architecture Guard Dashboard`** in the Merge Request, GitLab renders the interactive dashboard immediately in the browser.

---

#### Approach C: GitHub Actions Step Summary (Instant On-Screen Display)

GitHub Actions supports `$GITHUB_STEP_SUMMARY` to render markdown directly on the workflow run page:

```yaml
      - name: Render Summary in GitHub Actions Run
        if: always()
        run: |
          REVIEW_FILE=$(find . -name "code-review.md" -o -name "summary.md" | head -n 1)
          if [ -n "$REVIEW_FILE" ]; then
            cat "$REVIEW_FILE" >> $GITHUB_STEP_SUMMARY
          fi
```
Reviewers clicking the GitHub Actions build will see the full quality verdict, issues, and before/after fixes rendered directly on screen.

---

#### Approach D: Central Dashboard on GitHub Pages / GitLab Pages

To maintain a permanent, bookmarkable URL (e.g., `https://my-org.github.io/order-service/`) for your default branch:

* **In GitHub Actions:**
  ```yaml
        - name: Deploy to GitHub Pages
          if: github.ref == 'refs/heads/main'
          uses: peaceiris/actions-gh-pages@v4
          with:
            github_token: ${{ secrets.GITHUB_TOKEN }}
            publish_dir: target/architecture-guard
  ```

* **In GitLab CI/CD:**
  ```yaml
  pages:
    stage: deploy
    rules:
      - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    script:
      - mkdir -p public
      - cp $(find . -name "architecture-guard-report.html" | head -n 1) public/index.html
    artifacts:
      paths:
        - public
  ```

---

## 12. Operational Best Practices & Crash Prevention

To guarantee zero operational failures, prevent out-of-memory errors, and ensure seamless CI/CD execution across both tiny microservices and massive enterprise monoliths, adhere to the following operational best practices.

---

### 🚨 Best Practice #1: Right-Size JVM Heap Memory (CRASH & OOM PREVENTION)

> [!IMPORTANT]
> **Crash Vectors:** 
> 1. `java.lang.OutOfMemoryError: Java heap space` (Heap too small on large repositories).
> 2. `Linux OOM Killer (exit code 137)` (Heap configured larger than container RAM quota).

ArchGuard loads compiled class metadata and AST structures into memory to build the dependency graph. Sizing should match your project size and CI runner constraints:

#### Memory Sizing Matrix:

| Project Size | Production Classes | Recommended `MAVEN_OPTS` | Typical ArchGuard Heap Usage |
| :--- | :--- | :--- | :--- |
| **Small / Microservice** | < 500 classes | **Default Maven** (or `-Xmx1g`) | ~120 MB – 250 MB |
| **Medium Service** | 500 – 2,500 classes | **`-Xmx1g` to `-Xmx2g`** | ~350 MB – 800 MB |
| **Large Project** | 2,500 – 5,000 classes | **`-Xmx2g`** | ~1.0 GB – 1.6 GB |
| **Mega-Monolith** | 5,000 – 20,000+ classes | **`-Xmx3g` to `-Xmx4g`** | ~2.0 GB – 3.2 GB |

#### CI/CD Container Safety Rule:
> [!WARNING]
> If your CI/CD runner or Docker container has only **2 GB or 4 GB total RAM**, setting `-Xmx4g` will cause the Linux kernel to terminate the container via the OOM Killer (`exit code 137`).
> **Rule of Thumb:** Keep `-Xmx` at or below **75% of your container's total allocated RAM** (e.g. for a 2 GB container, use `-Xmx1536m`; for a 4 GB container, use `-Xmx3g`).

#### Sizing Configuration Examples:

* **For standard microservices & medium projects (Recommended for most teams):**
  ```bash
  export MAVEN_OPTS="-Xmx2g -XX:+UseG1GC"
  ```

* **For massive 5,000+ class enterprise monoliths:**
  ```bash
  export MAVEN_OPTS="-Xmx4g -XX:+UseG1GC"
  ```

* **In `.mvn/jvm.config` (Recommended for repo-level consistency):**
  ```properties
  -Xmx2g
  -XX:+UseG1GC
  -XX:+TieredCompilation
  ```

---

### 🚨 Best Practice #2: Always Execute Within the Lifecycle (`mvn clean verify`)

> [!IMPORTANT]
> **Crash Vector:** Missing bytecode directory or false `0 classes analyzed`
> ArchGuard inspects compiled production bytecode (`target/classes`) alongside source code (`src/main/java`). Running `mvn architecture-guard:check` standalone on a newly cloned, uncompiled repository will find no classes, logging a warning and safely skipping analysis.

#### The Solution:
1. **Never run raw `check` on uncompiled code.** By default, ArchGuard binds to the `prepare-package` lifecycle phase. This guarantees that:
   - All compiled classes (`target/classes`) and unit tests (`test`) are fully executed before analysis.
   - If any architectural violations fail the quality gate, Maven aborts the build **before** the `package` phase is reached—preventing an invalid or non-compliant `.jar` artifact from ever being built or deployed.
   - Any pre-existing JARs in `target/` are deleted upon failure.
   ```bash
   # Recommended full pipeline command
   mvn clean verify
   ```
2. **If running the plugin goal standalone**, ensure classes are compiled first:
   ```bash
   mvn test-compile architecture-guard:check
   ```
3. **In Multi-Module Reactor Projects:** Running `mvn verify` from the parent root ensures upstream sibling module JARs and classes are built into `target/classes` and installed in the local reactor before downstream dependent modules are analyzed.

---

### 🛡️ Additional Stability & Operational Hardening Guidelines

#### 3. Baseline Snapshot Permissions & Git Management
* Always commit `.architecture-guard-baseline.json` into source control so pull requests compare against the frozen master baseline.
* Ensure CI/CD runner agents have read permissions for the baseline file and write permissions for the output directory (`target/architecture-guard/`).
* If a baseline file ever becomes corrupted or truncated, ArchGuard will cleanly trap it and produce an explicit `[BASELINE_LOADER]` error rather than silently failing the build.

#### 4. Safe Ratchet Mode in CI (`-Darchguard.ratchet=true`)
* Enable automatic ratcheting on your **default branch (main/master)** builds to shrink technical debt as developers refactor code.
* On feature PR branches, leave ratchet disabled (`false`, the default) so PRs only block newly added debt without prematurely mutating the baseline file on unmerged branches.

#### 5. Fault-Tolerant AST Parsing for Java 21+
* ArchGuard automatically tolerates modern Java constructs including `record` declarations, `sealed` interfaces, and pattern matching.
* If individual files contain compilation or syntax errors, ArchGuard safely logs a `[SOURCE_PARSER]` finding for that file while continuing to analyze the rest of the repository without terminating the pipeline.

---

## 13. Troubleshooting & FAQ

#### Q1: Why did the build succeed even though a new violation appeared in the report?
**Answer:** ArchGuard defaults to `<failOnSeverity>ERROR</failOnSeverity>`. If your new violation is a `WARNING` (e.g. `System.out.println` or `@Autowired`), it will be marked as `NEW` in the report but will not break the build. To fail on warnings, set `<failOnSeverity>WARNING</failOnSeverity>` in your `pom.xml` or pass `-Darchguard.failOnSeverity=WARNING`.

#### Q2: What happens when I rename or move a class?
**Answer:** ArchGuard’s fingerprinting uses the fully qualified class name. If a class is renamed, any legacy baseline violations in the old class will be marked as `RESOLVED`, and violations in the new class name will be tracked as `NEW`. You can re-run `mvn architecture-guard:baseline` to update the snapshot.

#### Q3: How do I exclude generated classes (MapStruct, Lombok, OpenAPI)?
**Answer:** In `archguard.yml` / `architecture-guard.json`, add regex patterns to the `"exclusions"` list (e.g., `[".*Generated.*", ".*MapperImpl.*"]`), or use in-code `@SuppressArchitecture("ALL")`.

#### Q4: Does ArchGuard support Kotlin application code (`.kt`)?
**Answer:** No. ArchGuard is engineered specifically for **Java** application source code (`.java`) and bytecode (`.class`). While the Gradle plugin supports Gradle build configuration scripts written in **Kotlin DSL (`build.gradle.kts`)**, the application codebase being scanned must be Java. Scanning Kotlin language syntax (`.kt` files) is not supported.

---

### 💬 Need Help, Found a Bug, or Have a Feature Request?

If you encounter an issue not resolved above, have a problem with your build, or would like to suggest an improvement:

👉 **[Post an Issue on GitHub](https://github.com/javatechgroup/archGaurdDocs/issues)**  
**Repository:** [https://github.com/javatechgroup/archGaurdDocs](https://github.com/javatechgroup/archGaurdDocs)

Please include the following details in your issue to help us diagnose and provide a fast solution:
1. **Java & Build Tool Version:** (`java -version`, `mvn -v`, or `gradle -v`)
2. **Configuration Snippet:** Your `pom.xml` / `build.gradle` plugin declaration or `archguard.yml`
3. **Execution Logs / Stack Trace:** Full error logs (run with `mvn clean verify -X` if necessary)

*Our development team actively monitors all incoming issues and will work closely with you to provide a fix or guidance.*

---
*Maintained by the Architecture Guard Core Team. Built for Java 21 & Spring Boot.*

