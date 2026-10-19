# 🛡️ ArchGuard — Features & Competitive Landscape

<p align="center">
  <img src="https://img.shields.io/badge/Java-21%2B-ED8B00?style=for-the-badge&logo=openjdk&logoColor=white" alt="Java 21+" />
  <img src="https://img.shields.io/badge/Spring_Boot-3.x%20%7C%206.x-6DB33F?style=for-the-badge&logo=springboot&logoColor=white" alt="Spring Boot 3+" />
  <img src="https://img.shields.io/badge/Infrastructure-Zero_Server-007ACC?style=for-the-badge" alt="Zero Server" />
  <img src="https://img.shields.io/badge/Standard-OASIS_SARIF_2.1.0-6f42c1?style=for-the-badge" alt="SARIF 2.1.0" />
  <img src="https://img.shields.io/badge/Dual_Engine-ASM_Bytecode_%2B_JavaParser_AST-000000?style=for-the-badge" alt="Dual Engine" />
  <img src="https://img.shields.io/badge/Build_Tool-Apache_Maven-C71A36?style=for-the-badge&logo=apachemaven&logoColor=white" alt="Maven" />
</p>

> **The Next-Generation Architecture Governance, Spring Boot Intelligence & Zero-Infrastructure Code Quality Gate.**

---

## 📑 Table of Contents
1. [The "Why ArchGuard?" Problem Statement](#1-the-why-archguard-problem-statement)
2. [How It Works: Hybrid Dual-Engine Architecture](#2-how-it-works-hybrid-dual-engine-architecture)
3. [Top 5 "Killer Capabilities"](#3-top-5-killer-capabilities)
4. [Memory & Resource Safety: What ArchGuard Detects (and What It Doesn't)](#4-memory--resource-safety-what-archguard-detects-and-what-it-doesnt)
5. [Head-to-Head Comparison Matrix](#5-head-to-head-comparison-matrix)
6. [In-Depth Tool Battles](#6-in-depth-tool-battles)
   - [ArchGuard vs SonarQube / SonarCloud](#1-archguard-vs-sonarqube--sonarcloud)
   - [ArchGuard vs ArchUnit](#2-archguard-vs-archunit)
   - [ArchGuard vs SpotBugs / FindBugs](#3-archguard-vs-spotbugs--findbugs)
   - [ArchGuard vs Checkstyle & PMD](#4-archguard-vs-checkstyle--pmd)
7. [Complete 30-Rule Quality Catalog (AG001 – AG030)](#7-complete-30-rule-quality-catalog-ag001--ag030)
8. [Enterprise Developer Experience & Reporting](#8-enterprise-developer-experience--reporting)
9. [Enterprise Scalability & Performance Benchmarks](#9-enterprise-scalability--performance-benchmarks)
10. [Get Started in Under 60 Seconds](#10-get-started-in-under-60-seconds)
11. [Next-Gen Strategic Roadmap](#11-next-gen-strategic-roadmap)

---

## 1. The "Why ArchGuard?" Problem Statement

Every software team runs linters and quality gates. Yet, production outages, memory leaks, and architectural erosion still happen every single day in Spring Boot services:

```
❌ A @Transactional method calls another @Transactional method in the same class (proxy bypass)
   👉 Result: Transaction never commits; silent data corruption in production!

❌ A singleton @Service contains a non-final List or SimpleDateFormat
   👉 Result: Multi-threaded race conditions; User A sees User B's private account data!

❌ A @RestController returns a JPA @Entity directly instead of a DTO/Record
   👉 Result: Mass-assignment vulnerability, Cartesian N+1 DB explosion, LazyInitializationException!

❌ A team turns on a quality gate on a 5-year-old enterprise application
   👉 Result: 3,400 legacy violations trigger build failures; developers revolt and disable the tool!
```

### Why Existing Tools Leave You Exposed:
- **Linters (Checkstyle, PMD):** Only inspect text/syntax. They don't understand bytecode, type hierarchies, or Spring proxies.
- **Bug Finders (SpotBugs):** Only inspect isolated bytecode. They have zero awareness of layered architecture or Spring bean scopes.
- **Enterprise Suites (SonarQube):** Require heavy server installations, databases, cloud licensing, network latency, and miss Spring-specific runtime traps.
- **Testing Libraries (ArchUnit):** Require writing and maintaining hundreds of lines of imperative JUnit test code with zero built-in differential debt ratcheting.

### 💡 The ArchGuard Solution:
**ArchGuard** is built from the ground up specifically for **Modern Java 21+ and Spring Boot 3+**. It combines **ASM Bytecode Analysis** with **JavaParser AST Inspection** into a single, lightning-fast Maven plugin with zero server infrastructure, automated ratcheting baselines, and native GitHub SARIF integration.

---

## 2. How It Works: Hybrid Dual-Engine Architecture

ArchGuard executes directly in the Maven `verify` lifecycle in milliseconds without sending code over the network:

```mermaid
flowchart TD
    classDef inputNode fill:#e0f2fe,stroke:#0284c7,stroke-width:2px,color:#0369a1;
    classDef engineNode fill:#f0fdf4,stroke:#16a34a,stroke-width:2px,color:#15803d;
    classDef govNode fill:#fef3c7,stroke:#d97706,stroke-width:2px,color:#b45309;
    classDef outNode fill:#f3e8ff,stroke:#9333ea,stroke-width:2px,color:#7e22ce;
    classDef passNode fill:#dcfce7,stroke:#16a34a,stroke-width:2px,color:#15803d;
    classDef failNode fill:#fee2e2,stroke:#dc2626,stroke-width:2px,color:#991b1b;

    subgraph Input["Your Java 21+ Project"]
        SRC["Java Source Code (.java)"]:::inputNode
        CLS["Compiled Bytecode (.class)"]:::inputNode
    end

    subgraph DualEngine["ArchGuard Dual-Engine Analyzer"]
        AST["JavaParser AST Engine\n• Semantic Code Smells\n• Spring Annotations\n• Secrets & Exception Hygiene"]:::engineNode
        ASM["ASM High-Speed Bytecode Engine\n• Layer Dependency Hierarchy\n• Cyclic Graphs (Tarjan's SCC)\n• Type & Call Graph Resolution"]:::engineNode
    end

    subgraph Governance["Governance & Baselining Gate"]
        DIFF["Differential Snapshot Engine\n(.architecture-guard-baseline.json)\n• Track SHA-256 Fingerprints\n• Multiset Frequency Protection\n• Auto-Ratcheting (Debt Shrinker)"]:::govNode
    end

    subgraph Output["Zero-Infrastructure Outputs"]
        HTML["Interactive HTML Dashboard\n(PR Delta / Baselined / All Code)"]:::outNode
        SARIF["OASIS SARIF 2.1.0\n(GitHub PR Annotations & Security Tab)"]:::outNode
        GATE{"Quality Gate Threshold\n(failOnSeverity = ERROR)"}:::govNode
    end

    SRC --> AST
    CLS --> ASM
    AST --> DIFF
    ASM --> DIFF
    DIFF --> HTML
    DIFF --> SARIF
    DIFF --> GATE
    GATE -->|New Violations Found| FAIL["❌ BUILD FAILED"]:::failNode
    GATE -->|Clean PR Delta| PASS["✅ BUILD SUCCESS"]:::passNode
```

---

## 3. Top 5 "Killer Capabilities"

### 🔥 1. Spring AOP Proxy Self-Invocation Hunter (`AG021`)
In Spring, `@Transactional`, `@Async`, `@Cacheable`, `@Retryable`, and `@Scheduled` work via dynamic AOP proxies. When a method calls another annotated method in the **same class** (`this.method()`), Spring proxies are bypassed silently.

```java
// ❌ SILENT PRODUCTION DISASTER: The @Transactional proxy is bypassed!
@Service
public class OrderService {
    public void processAll(List<Order> orders) {
        for (Order order : orders) {
            saveWithAudit(order); // Calls directly via 'this', bypassing Spring proxy!
        }
    }

    @Transactional // NEVER OPENS A TRANSACTION WHEN CALLED FROM processAll()!
    public void saveWithAudit(Order order) {
        orderRepo.save(order);
        auditRepo.record(order);
    }
}
```
> **ArchGuard catches this instantly during `mvn compile` and suggests extracting the transactional boundary into an injected bean or programmatic `TransactionTemplate`.**

---

### 🔥 2. Singleton Bean Concurrency & Mutable State Guard (`AG022`)
Spring Beans are singletons shared across concurrent HTTP request worker threads. Defining mutable instance state or non-thread-safe utilities in `@Service` or `@Component` leads to catastrophic data leaks.

```java
// ❌ RACE CONDITION: Shared across all concurrent HTTP requests!
@Service
public class PaymentProcessingService {
    private List<Payment> recentPayments = new ArrayList<>(); // Corrupted under concurrency!
    private SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd"); // Throws NumberFormatException / cross-talk!
    private int transactionCounter = 0; // Lost updates!
}
```
> **ArchGuard flags non-final mutable fields in Spring singletons and mandates thread-safe alternatives (`java.time.DateTimeFormatter`, scoped state, or concurrent collections).**

---

### 🔥 3. Zero-Fatigue Differential Baselining with Auto-Ratcheting
Adopting quality gates in large enterprise codebases usually fails because developers are overwhelmed by thousands of legacy warnings they didn't write.

```bash
# Day 1: Freeze all legacy technical debt in 1 command
mvn architecture-guard:baseline
```

1. **Only New Debt Fails the Build:** PRs only fail if the developer adds **new** violations.
2. **Multiset Protection:** If a legacy class has 1 warning, adding a 2nd warning in the same file is recognized as **new** debt and blocked!
3. **Automatic Ratchet (`-Darchguard.ratchet=true`):** As developers refactor and clean up old code, the baseline snapshot shrinks automatically. **Technical debt can only decrease, never increase.**
4. **Two-Pass Semantic Matching (Zero-Drift Baseline):** Unlike naive tools whose baselines break whenever lines shift or methods are edited, ArchGuard pairs exact SHA-256 fingerprints (Pass 1) with semantic identity keys (Pass 2):
   - **Metric Fluctuation Resilience:** Adding lines inside a baselined monster method (`AG020`), adding branches to a complex method (`AG018`), or injecting collaborators (`AG017`) will **never** trigger false "new violations" or false "resolved debt."
   - **Refactoring Resilience:** Renaming variables, `@RequestBody` parameters (`AG015`), exception catch variables (`e` → `ex` in `AG008`), or moving code lines around preserves existing baseline identity.

---

### 🔥 4. REST API & Domain Boundary Firewall (`AG001`, `AG012`, `AG015`, `AG016`)
Protects your application's external API boundary and internal domain model:

```java
// ❌ ARCHITECTURAL BREACH & SECURITY VULNERABILITY
@RestController
public class UserController {
    @Autowired
    private UserRepository userRepository; // AG001: Controller directly touches DB!

    @GetMapping("/users")
    public List<UserEntity> getAllUsers() { // AG012: JPA Entity exposed! AG016: Unpaginated List!
        return userRepository.findAll();
    }

    @PostMapping("/users")
    public UserEntity createUser(@RequestBody UserEntity user) { // AG015: Missing @Valid!
        return userRepository.save(user);
    }
}
```
> **ArchGuard enforces clean architecture: Controllers talk only to Services, expose Java 21 Records/DTOs, enforce `@Valid` request bodies, and return paginated collections (`Page<T>`).**

---

### 🔥 5. Zero-Infrastructure Native GitHub SARIF 2.1.0 & HTML Dashboard
No server to host, no license keys, no Docker containers:
- **GitHub PR Inline Annotations:** Generates standard OASIS SARIF 2.1.0 uploaded via `github/codeql-action/upload-sarif@v3`. Annotations appear directly on the changed lines of your GitHub Pull Request.
- **Security Tab Integration:** All architecture and security violations populate the native **GitHub Security → Code Scanning** dashboard for free on public repositories.
- **Offline Interactive HTML Dashboard:** Self-contained report with zero CDN dependencies, live filtering by scope (**PR Delta**, **Baselined Debt**, **All Code**), dark/light theme, and copy-paste remediation code.

---

## 4. Memory & Resource Safety: What ArchGuard Detects (and What It Doesn't)

To maintain absolute technical integrity, here is an exact, transparent breakdown of how ArchGuard addresses memory and resource hazards:

### ✅ What ArchGuard Handles (Static Code Root-Cause Analysis)
ArchGuard detects the **static code patterns and architectural smells** that directly trigger JVM memory leaks, native resource exhaustion, and `OutOfMemoryError` (OOM) crashes in production:

1. **Unbounded Static Memory Accumulation (`AG010`):**
   - Non-final static fields or mutable collections (`static List<T>`, `static Map<K,V>`).
   - *Why it matters:* In the JVM, static variables are attached directly to classloader GC roots and **never garbage-collected**, creating permanent memory leaks across HTTP requests.
2. **Spring Singleton Bean Memory Retention (`AG022`):**
   - Mutable instance collections and non-thread-safe state inside `@Service`, `@Component`, or `@RestController` beans.
   - *Why it matters:* Because Spring singleton beans live for the entire application lifecycle, accumulating data in instance fields causes unbounded heap growth.
3. **OS File Descriptor & Connection Pool Leaks (`AG024`):**
   - Instantiating `AutoCloseable` streams (`FileInputStream`, `FileOutputStream`, `BufferedReader`) or JDBC resources (`Connection`, `Statement`, `ResultSet`) outside a **try-with-resources** block.
   - *Why it matters:* Exceptions before an explicit `.close()` leak OS file handles, sockets, and pool connections, leading to crash states.
4. **JVM OutOfMemoryError via Unbounded Database Queries (`AG016`):**
   - Returning raw `List<T>` from REST endpoints instead of `Page<T>` or slice pagination.
   - *Why it matters:* Fetching 100,000+ records in a single HTTP request forces the JVM to materialize massive object graphs in heap, causing sudden `java.lang.OutOfMemoryError: Java heap space`.
5. **Garbage Collection (GC) Thrashing from String Concatenation (`AG025`):**
   - Using `+` or `+=` inside loops (`for`, `while`).
   - *Why it matters:* Repeated concatenation creates $N$ intermediate immutable `String` instances on every iteration, leading to $O(n^2)$ heap allocation and frequent Stop-The-World GC pauses.
6. **Cartesian Product Heap Bloat (`AG014`):**
   - Using JPA `FetchType.EAGER` relationships.
   - *Why it matters:* Eager joins trigger Cartesian product query returns from the database, loading thousands of redundant entities into the JVM heap.
7. **ThreadLocal Memory Leaks & Worker Contamination (`AG028`):**
   - Calling `ThreadLocal.set()` without guaranteed `.remove()` in an enclosing `finally` block.
   - *Why it matters:* Web containers (Tomcat, Jetty, Undertow) recycle worker threads from fixed thread pools. Uncleaned `ThreadLocal` entries remain pinned to `Thread` objects indefinitely, leading to classloader memory leaks and cross-request user/tenant context leaks.
8. **Unbounded Thread Allocation & Native Thread Starvation (`AG029`):**
   - Using `@Async` without an explicit `ThreadPoolTaskExecutor` qualifier.
   - *Why it matters:* Defaulting to `SimpleAsyncTaskExecutor` spawns an unbounded new OS thread for every single asynchronous task call, triggering `java.lang.OutOfMemoryError: unable to create new native thread` under load.

### ❌ What ArchGuard Does NOT Handle (Transparent Boundaries)
ArchGuard is a **static analysis code gate**, not a runtime JVM profiler:
- **No Dynamic Heap Dump Analysis:** ArchGuard does not inspect live JVM heap dumps (`.hprof`), JVM flags (`-Xmx`, `-Xms`), or live object retention trees. (Use tools like *Eclipse MAT* or *YourKit* for post-mortem heap dump profiling).
- **No Live Memory Allocation Profiling:** ArchGuard cannot measure the dynamic runtime byte size of objects in flight. (Use *JProfiler* or *VisualVM* for runtime telemetry).
- **No Native Memory Profiling:** ArchGuard does not track off-heap JNI/C++ allocations.

---

## 5. Head-to-Head Comparison Matrix

| Capability / Feature | 🛡️ **ArchGuard** | 🏢 **SonarQube** | 📐 **ArchUnit** | 🐛 **SpotBugs** | 🎨 **Checkstyle / PMD** |
| :--- | :---: | :---: | :---: | :---: | :---: |
| **Server / Cloud Infrastructure** | **None (100% Embedded)** | Heavy Server + DB | None (JUnit tests) | None (Plugin) | None (Plugin) |
| **Execution Speed** | **~2–7 seconds (In-Process)** | Minutes (Server queue + DB) | Test suite overhead | Fast (~5–15s) | Fast (~5–10s) |
| **Cost & Licensing** | **100% Free & Open Source** | Expensive Enterprise Tier | Free & Open Source | Free & Open Source | Free & Open Source |
| **Spring Proxy Self-Invocation (`AG021`)** | ✅ **Built-in Native** | ❌ None | ⚠️ Custom test code | ❌ None | ❌ None |
| **Spring Singleton Race Conditions (`AG022`)**| ✅ **Built-in Native** | ⚠️ Static only | ⚠️ Custom test code | ⚠️ Static only | ❌ None |
| **Direct JPA Entity REST Leakage (`AG012`)** | ✅ **Built-in Native** | ❌ None | ⚠️ Custom test code | ❌ None | ❌ None |
| **Architectural Layer Enforcement** | ✅ **Declarative YAML** | ⚠️ Enterprise only | ✅ Java DSL unit tests| ❌ None | ⚠️ Basic imports |
| **Cyclic Dependency Detection (Cycles)** | ✅ **Package & Class** | ✅ Project level | ✅ Package level | ❌ None | ❌ None |
| **Missing `@Transactional` Boundary (`AG009`)**| ✅ **Built-in Native** | ❌ None | ⚠️ Custom test code | ❌ None | ❌ None |
| **JPA Eager Fetch / N+1 Risk (`AG014`)** | ✅ **Built-in Native** | ❌ None | ⚠️ Custom test code | ❌ None | ❌ None |
| **Differential Baselining (PR Delta)** | ✅ **Multiset Snapshot** | ✅ New Code Period | ❌ None | ⚠️ XML Filter | ⚠️ Suppressions |
| **Automatic Debt Ratcheting** | ✅ **Built-in Mode** | ❌ Manual tuning | ❌ None | ❌ None | ❌ None |
| **Baseline Refactor Resilience** | ✅ **Two-Pass Semantic (Zero-Drift)** | ⚠️ Fragile Git-diff shifts | ❌ No baseline | ❌ Breaks on line shifts | ❌ Breaks on line shifts |
| **Interactive Offline HTML Dashboard** | ✅ **Included (Zero CDN)**| ❌ Web server only | ❌ None | ⚠️ Basic static | ⚠️ Basic static |
| **GitHub Security / SARIF 2.1.0** | ✅ **Native Output** | ⚠️ Plugin / Action | ❌ None | ⚠️ Via plugin | ⚠️ Via plugin |
| **Setup Time** | **< 60 Seconds (Zero-Config)** | Hours / Days | Hours writing tests | 10 Minutes | 10 Minutes |

---

## 6. In-Depth Tool Battles

### 1. ArchGuard vs SonarQube / SonarCloud

| Comparison Vector | 🛡️ ArchGuard | 🏢 SonarQube |
| :--- | :--- | :--- |
| **Infrastructure Overhead** | **Zero.** Executes directly inside your Maven build. No Docker containers, no PostgreSQL instance, no webhook management. | **High.** Requires provisioning and maintaining an application server, dedicated database, network firewalls, and tokens. |
| **Spring Boot Awareness** | **Engineered for Spring Boot 3+.** Understands proxy boundaries, singleton thread hazards, `@Transactional` boundaries, and REST best practices. | **Generic Java.** Treats Spring like any plain Java code. Completely misses proxy self-invocation and REST entity exposure. |
| **Feedback Loop Speed** | **Immediate (Shift-Left).** Delivered in-process in your local terminal during `mvn verify` in ~2–7 seconds. | **Asynchronous.** Code bundle uploaded to server; results available after server worker queue processing and DB indexing. |
| **Cost & PR Decoration** | **100% Free.** Generates SARIF 2.1.0 reports that natively decorate PR lines in GitHub Actions for free. | **Paywalled.** Pull request decoration and branch analysis are locked behind paid Developer/Enterprise editions. |

> **Verdict:** Use **ArchGuard** as your zero-friction, Spring-intelligent gatekeeper directly in local builds and PR pipelines (shift-left). Use SonarQube for broad multi-language portfolio tracking and enterprise reporting if you already host the centralized infrastructure.

---

### 2. ArchGuard vs ArchUnit

| Comparison Vector | 🛡️ ArchGuard | 📐 ArchUnit |
| :--- | :--- | :--- |
| **Configuration vs Coding** | **Zero-Config & Declarative.** Works out of the box with zero files. Optional `archguard.yml` (via `mvn architecture-guard:init`) for custom overrides. Zero test code needed. | **Programmatic.** Developers must write, compile, and maintain verbose JUnit unit tests (`classes().that().resideInAPackage()...`). |
| **Legacy Technical Debt** | **Built-in Ratchet Engine.** Freeze legacy debt with `mvn architecture-guard:baseline` in 1 second. | **No native baseline.** Teams must either fix all legacy violations before adding a rule or write complex custom ignore logic. |
| **Rule Breadth** | Covers architecture, Spring runtime traps, clean code metrics, secrets, and performance anti-patterns out of the box. | Strictly limited to architectural dependency and reflection assertions. |
| **Reporting & Dashboards** | Rich interactive HTML dashboard + SARIF 2.1.0 export. | Standard JUnit assertion failure messages in unit test logs. |

> **Verdict:** ArchGuard provides a complete out-of-the-box governance product with UI, baselines, and SARIF, whereas ArchUnit is a testing library requiring custom code.

---

### 3. ArchGuard vs SpotBugs / FindBugs

| Comparison Vector | 🛡️ ArchGuard | 🐛 SpotBugs |
| :--- | :--- | :--- |
| **Context & Architecture** | Analyzes full architectural layer graphs, package boundaries, and Spring component lifecycles. | Analyzes low-level bytecode patterns in isolation without architectural context. |
| **Developer Experience** | Modern YAML configuration, clean terminal diffs, and an interactive modern HTML UI. | Complex XML suppression filters and 2000s-era static HTML reports. |

---

### 4. ArchGuard vs Checkstyle & PMD

| Comparison Vector | 🛡️ ArchGuard | 🎨 Checkstyle & PMD |
| :--- | :--- | :--- |
| **Analysis Depth** | Dual-Engine: ASM Bytecode + AST Source. Resolves actual class hierarchies and cross-package references. | Source AST and text pattern matching only. Cannot resolve true compiled bytecode types. |
| **Focus Area** | Framework safety, architectural integrity, and runtime concurrency bugs. | Code style, indentation, naming conventions, and cosmetic formatting. |

---

## 7. Complete 30-Rule Quality Catalog (AG001 – AG030)

Every rule includes automated diagnostic detection, exact line numbers, and actionable remediation suggestions:

### 🏛️ Category 1: Architecture & Modular Boundaries
- **`[AG001]` Controller to Repository Direct Dependency (`ERROR`):** Enforces service layer orchestration; forbids controllers from bypassing service logic to touch DB directly.
- **`[AG002]` Controller to Controller Direct Dependency (`ERROR`):** Forbids web controllers from depending on or calling other controllers directly.
- **`[AG003]` Service to Controller Inverted Dependency (`ERROR`):** Prohibits low-level or business service layer from depending upwards on presentation controllers.
- **`[AG004]` Repository to Service Inverted Dependency (`ERROR`):** Forbids persistence layer components from coupling upwards to service orchestration.
- **`[AG005]` Class-Level Cyclic Dependency (`ERROR`):** Detects cyclic dependency loops (A $\to$ B $\to$ A) using Tarjan's Strongly Connected Components algorithm.
- **`[AG006]` Package Boundary Violation (`ERROR`):** Enforces modular encapsulation and package access restrictions between bounded contexts.

### 🌱 Category 2: Spring Boot & Concurrency Hazards
- **`[AG007]` Field Injection (`@Autowired`) (`WARNING`):** Mandates constructor injection for immutability and easier unit testing.
- **`[AG009]` Missing `@Transactional` on Mutating Methods (`WARNING`):** Flags public service methods executing repository writes without transaction boundaries.
- **`[AG010]` Unsafe Mutable Static State (`ERROR`):** Forbids mutable static fields/collections that corrupt state in multi-threaded server runtimes.
- **`[AG021]` Spring Self-Invocation Proxy Bypass (`ERROR`):** Detects internal `this.method()` calls that silently bypass `@Transactional`, `@Async`, or `@Cacheable` proxies.
- **`[AG022]` Singleton Bean Mutable Instance State (`ERROR`):** Prevents race conditions and cross-request data leaks caused by mutable state in `@Service` singletons.
- **`[AG026]` Remote I/O in `@Transactional` Boundary (`ERROR`):** Prevents HikariCP connection pool starvation and outages caused by remote HTTP/REST/SDK calls inside transactions.
- **`[AG027]` Java 21 Virtual Thread Pinning (`WARNING`):** Detects `synchronized` methods/blocks that pin OS carrier threads in Project Loom, degrading virtual thread throughput.
- **`[AG028]` ThreadLocal Memory Leak & Context Contamination (`ERROR`):** Flags `ThreadLocal.set()` without guaranteed `remove()` in a `finally` block to protect pooled worker threads.
- **`[AG029]` Unbounded `@Async` Thread Pool Exhaustion (`WARNING`):** Prevents native OS thread starvation by ensuring `@Async` methods declare a bounded `ThreadPoolTaskExecutor`.

### 🌐 Category 3: REST API & Data Governance
- **`[AG012]` Direct Entity Exposure in REST API (`ERROR`):** Forbids JPA `@Entity` classes in REST controller endpoints, preventing mass-assignment and lazy-loading crashes.
- **`[AG014]` JPA Eager Fetch / N+1 Query Risk (`WARNING`):** Prohibits `FetchType.EAGER` to prevent Cartesian explosion SQL queries.
- **`[AG015]` Missing `@Valid` on REST `@RequestBody` (`WARNING`):** Ensures all mutation endpoints enforce Bean Validation constraints.
- **`[AG016]` Unpaginated Collections in REST APIs (`WARNING`):** Enforces `Page<T>` returns instead of unbounded `List<T>` to prevent OOM errors.

### 🔒 Category 4: Security, Resilience & Safety
- **`[AG008]` Swallowed Exception Blocks (`WARNING`):** Forbids empty catches or caught exceptions that are neither logged nor rethrown.
- **`[AG013]` Hardcoded Secrets Scanner (`ERROR`):** High-entropy scanner detecting committed credentials, API tokens, and AWS keys.
- **`[AG023]` NullPointer Traps & Defensive Coding (`WARNING`):** Flags raw `Optional.get()`, methods returning `null` instead of empty collections, and inverted equality checks.
- **`[AG024]` Unclosed AutoCloseable Resource Leaks (`ERROR`):** Enforces try-with-resources for I/O streams, readers, and JDBC resources.
- **`[AG030]` Missing HTTP Client Timeout Configuration (`ERROR`):** Eliminates worker thread starvation by ensuring `RestTemplate`, `RestClient`, and `HttpClient` configure explicit connection and read timeouts.

### ⚡ Category 5: Performance & Clean Code
- **`[AG011]` Direct Console Logging (`WARNING`):** Eliminates `System.out.println` and `e.printStackTrace()` in favor of structured logging.
- **`[AG017]` Injected Dependency Bloat / God Class (`WARNING`):** Flags classes with $\ge 7$ injected collaborators violating Single Responsibility.
- **`[AG018]` High Cyclomatic Complexity (`WARNING`):** Enforces clean, testable code by flagging methods with complexity $> 15$.
- **`[AG019]` Algorithmic Performance Risk (`WARNING`):** Flags linear collection searches ($O(n)$) or database queries inside loops.
- **`[AG020]` Code Readability & Arrow Anti-patterns (`WARNING`):** Flags monster methods ($> 60$ lines), bloated files ($> 450$ lines), and deep nesting ($\ge 4$ levels).
- **`[AG025]` Loop String Concatenation (`WARNING`):** Flags `+` or `+=` string concatenation inside loops that cause $O(n^2)$ heap allocation churn.

---

## 8. Enterprise Developer Experience & Reporting

### 📊 1. Interactive Offline HTML Dashboard
Generated at `target/architecture-guard/report.html`:
- **Self-Contained & Zero CDN:** 100% standalone single-file HTML report with embedded CSS/JS that renders completely offline inside air-gapped enterprise environments.
- **Rule Sidebar "Issues Only" Toggle:** Reviewers can toggle between **`⚠️ Issues Only`** (shows only the subset of rules with active findings) and **`All Rules`** (full 30-rule catalog), with live rule search.
- **Auditable Suppressed Scope:** Dedicated **`🛡️ Suppressed (N)`** tab displays code-suppressed violations with distinct **`🛡️ SUPPRESSED IN CODE`** badges without polluting the active quality gate numbers.
- **One-Click IDE Deep Links:** Normalized links (`vscode://file/...` and `idea://open?file=...`) open the exact file and line directly in VS Code or IntelliJ IDEA on Windows, macOS, and Linux.
- **Zero-Results Filter Empty State:** When filter combinations yield zero hits, a clean feedback box with **`↺ Reset All Filters`** restores the view instantly.
- **Dynamic Scope Filters:** Toggle between **Active**, **PR Delta** (new violations only), **Baselined Debt**, and **Suppressed** with instant metric updates.
- **Actionable Remediation Cards:** Every violation displays the exact file, line number, offending code snippet, and copyable remediation guidance.
- **Dark / Light Theme:** Seamless toggle with persistent local preference.

### 🐙 2. Native GitHub Actions PR Decoration
Upload standard OASIS SARIF 2.1.0 in just 3 lines of YAML:

```yaml
- name: Upload ArchGuard SARIF Report
  uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: target/architecture-guard/report.sarif
    category: archguard
```

- **Inline Code Comments:** GitHub automatically places red warnings and error annotations on the exact lines of code modified in the Pull Request.
- **Security Dashboard:** Violations appear natively under the **Security → Code Scanning alerts** tab.

### 🏷️ 3. Auditable In-Code Suppressions (Method & Class Level)
Suppress intentional architecture exceptions using either standard Java `@SuppressWarnings` or `@SuppressArchitecture`. Suppressed violations **do not break CI builds**, but remain 100% visible and auditable under the `🛡️ Suppressed` dashboard tab:

```java
// Method-Level Targeted Suppression (Recommended)
@SuppressWarnings({"archguard:AG018", "archguard:AG020"})
public void legacyBatchProcess() { ... }

// Method-Level Blanket Suppression (All ArchGuard Rules)
@SuppressWarnings("archguard")
public void uninspectedLegacyMethod() { ... }

// Class-Level Suppression with Audit Rationale
@SuppressArchitecture(
    rule = "AG024",
    reason = "Resource lifecycle is managed externally by Spring Batch framework"
)
public class LegacyExportService { ... }
```

### 📦 4. Multi-Module Reactor & Parent POM Support
- **Upward Config Discovery:** Child submodules automatically walk up the directory tree (up to 4 levels) to discover the root `archguard.yml`, eliminating config duplication.
- **Aggregator Clean Skip:** Modules with `<packaging>pom</packaging>` are skipped cleanly without false warnings about missing `target/classes`.

---

## 9. Enterprise Scalability & Performance Benchmarks

In enterprise CI/CD pipelines, build speed directly impacts developer flow. Lengthy verification phases introduce pipeline friction and frequently cause engineering teams to disable quality gates. ArchGuard is architected from the ground up to minimize CI overhead, operating in-process within the standard Maven build lifecycle.

### ⏱️ Indicative Performance Profile: 1,000 Classes (~250,000 LOC)
*Representative timings observed on an 8-core developer workstation / standard 2-vCPU CI runner with SSD storage:*

| Analysis Phase | Underlying Engine | Latency (1,000 Classes) | Algorithmic Complexity | Engineering Design Note |
| :--- | :--- | :---: | :---: | :--- |
| **Bytecode Ingestion & Hierarchy** | **ASM / ArchUnit** | **~1.5 – 2.5s** | $O(N)$ | Single-pass bytecode parsing across compiled `.class` files |
| **Semantic Source AST Inspection** | **JavaParser AST** | **~2.5 – 4.5s** | $O(N)$ | Streaming file-by-file visitor with immediate AST deallocation |
| **Cyclic Dependency Resolution** | **Tarjan's SCC Graph** | **< 50ms** | $O(V + E)$ | Strictly linear strongly-connected-components traversal |
| **30 Quality & Spring Rules** | **ArchGuard Core** | **~150ms** | $O(1)$ lookups | In-memory evaluation over pre-computed type & AST metadata |
| **Differential Baseline & Ratchet** | **SHA-256 Multiset Engine** | **< 50ms** | $O(V)$ | Streaming hash-map fingerprint comparison against baseline snapshot |
| **Report Generation (HTML & SARIF)**| **Offline File Exporter** | **~200ms** | $O(V)$ | Direct streaming local file serialization (zero external HTTP calls) |
| **Total Wall-Clock Analysis Time** | | **~4.5 – 7.5 seconds** | — | ⚡ **Sub-10s turnaround in typical pull-request workflows** |

> [!NOTE]
> **Benchmark Variability:** Actual execution durations depend on CPU core count, disk I/O throughput (SSD vs. networked CI volume), JVM warmup state, and repository size. For microservices (< 300 classes), execution typically completes in under 2 seconds.

---

### 📊 Architectural Comparison: Local In-Process Gate vs. Centralized Platforms

Static analysis tools serve distinct architectural roles across the software lifecycle. Below is a representative comparison of typical wall-clock turnaround times experienced by developers during pull request verification:

```
Representative CI Feedback Turnaround (1,000 Classes, Standard CI Runner)
─────────────────────────────────────────────────────────────────────────────
ArchGuard (In-Process Maven Gate)        ██ 5.5s
PMD / Checkstyle (Local Source Linter)   ███ 8.0s
SpotBugs (Local Bytecode Analyzer)       ███████ 22.0s
Centralized Server Platform (SonarQube)*  ██████████████████████████████ ~180s (3m)
─────────────────────────────────────────────────────────────────────────────
*Note: Representative end-to-end turnaround including artifact bundling, network transfer,
worker queue scheduling, and centralized server-side database persistence.
```

#### 🔍 Benchmark Methodology & Architectural Context

When evaluating performance across analysis tools, it is essential to consider differences in architecture and execution scope:

1. **Why ArchGuard Operates in Low Single-Digit Seconds:**
   - **Zero-Network, In-Process Execution:** ArchGuard executes directly inside the local JVM process of `mvn verify`. There are no external HTTP transfers, no remote webhook handshakes, and no database roundtrips.
   - **Targeted Spring & Architectural Rules:** Rather than running hundreds of generic AST queries, ArchGuard focuses specifically on architectural boundaries, Spring runtime traps, and high-risk concurrency/resource hazards.
   - **Streaming AST Lifecycle:** Source files are parsed, evaluated, and immediately dereferenced for garbage collection, avoiding cumulative AST graph heap bloat.

2. **Context on Centralized Platforms (e.g., SonarQube):**
   - Centralized platforms are designed for comprehensive cross-repository portfolio management, historical tracking, multi-language coverage, and centralized rule administration.
   - Their longer end-to-end turnaround is primarily driven by operational scope: uploading source bundles to a central server, awaiting worker queue capacity, executing server-side analysis workers, and writing metrics to relational databases.
   - **Recommended Synergy:** ArchGuard is designed to complement existing centralized suites by acting as an **immediate, zero-cost shift-left quality gate** directly in developer terminals and pull request pipelines—providing sub-10s feedback before code ever reaches centralized scanners.

---

### 🧠 Zero-OOM Memory Safety (Heap Footprint)
Enterprise pipelines often fail with `java.lang.OutOfMemoryError` when linters retain massive syntax trees in memory:
- **Transient AST Lifecycle (Zero-Retention):** ArchGuard never retains all 1,000 AST models simultaneously. Each source file is parsed, evaluated for findings, and immediately garbage-collected.
- **Ultra-Compact Bytecode Symbols:** ASM bytecode representations consume only ~30–45 KB per class (~35 MB for 1,000 classes).
- **Peak Heap Consumption:** Peak memory usage is **under 300 MB**, running comfortably within standard Maven container limits (`-Xmx512m` or `-Xmx1024m`) without any custom JVM tuning.

---

## 10. Get Started in Under 60 Seconds

ArchGuard embraces a **Zero-Configuration** philosophy. It runs directly within your Maven build without requiring any configuration files, external daemons, or cloud tokens.

### Step 1: Add ArchGuard to `pom.xml` (Required)
Add the plugin to your project's `<build><plugins>` section. By default, all 30 rules are active and layers are auto-classified via Spring annotations:

```xml
<plugin>
    <groupId>io.github.javatechgroup</groupId>
    <artifactId>architecture-guard-maven-plugin</artifactId>
    <version>1.0.2</version>
    <executions>
        <execution>
            <goals>
                <goal>check</goal>
            </goals>
        </execution>
    </executions>
    <configuration>
        <failOnSeverity>ERROR</failOnSeverity>
    </configuration>
</plugin>
```

### Step 2: Freeze Legacy Debt (For Existing Projects)
If introducing ArchGuard into a large, existing codebase with legacy violations, freeze them immediately so existing debt never breaks the build:

```bash
mvn architecture-guard:baseline
```
*(Creates `.architecture-guard-baseline.json`. Only newly introduced violations in upcoming commits will fail the build).*

### Step 3: Run Analysis & Enforce Quality Gate
```bash
mvn clean verify
```
Reports are automatically generated offline in `target/architecture-guard/`:
- **`architecture-guard-report.html`**: Interactive self-contained developer dashboard
- **`report.sarif`**: Native OASIS SARIF 2.1.0 report for GitHub/GitLab PR scanning

---

### 💡 (Optional) Scaffold Custom Configuration (`archguard.yml`)
If your project uses non-standard architecture (e.g. Hexagonal / Clean Architecture) or you want to customize thresholds and rule severities, scaffold an initial configuration file:

```bash
mvn architecture-guard:init
```
*(Auto-detects your base package and writes a commented `archguard.yml` to your project root).*

---

## 11. Next-Gen Strategic Roadmap

ArchGuard is actively evolving to lead the Java 21+ and cloud-native ecosystem.

### ✅ Newly Delivered in Latest Engine:
* **`[AG026]` Remote I/O Inside `@Transactional` (HikariCP Pool Starvation Prevention)**: Delivered! Flags remote HTTP, REST, Feign, and cloud calls holding open database connections.
* **`[AG027]` Java 21 Virtual Thread Pinning (`synchronized` vs `ReentrantLock`)**: Delivered! Scans for carrier-thread-pinning `synchronized` blocks in Project Loom.

### 🔮 Future Capabilities Roadmap
* **Actionable Remediation Playbooks & Patch Generation (`mvn architecture-guard:patch`):**
  - Zero-risk developer-in-the-loop remediation: generates reviewable diff patches and copy-pasteable before/after code recipes for mechanical violations (field `@Autowired` to constructor injection, structured logging, and null-safe equality) without risking silent code breakages or circular dependency startup traps.
* **GitHub Actions Interactive PR Sticky Comment Bot:**
  - Automatically publishes and updates an interactive Markdown quality gate scorecard directly into the GitHub PR conversation thread.
* **Multi-Module Enterprise Aggregator:**
  - Cross-submodule boundary enforcement and unified aggregate HTML & SARIF reporting for massive multi-module enterprise Maven parent projects.

---

<p align="center">
  <b>Built with ❤️ for the Java & Spring Boot Community.</b><br/>
  Zero Infrastructure • Zero Network Overhead • Zero Developer Fatigue
</p>
