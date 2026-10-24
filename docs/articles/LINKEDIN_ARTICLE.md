# Stop Architectural Decay in Spring Boot Applications Before It Hits Production

**By Java Tech Group**  
*Tags: #Java #SpringBoot #SoftwareArchitecture #Maven #OpenSource #DevOps #Java21*

---

## 1. The Invisible Killers in Enterprise Java Codebases

Every engineering team starts a new Spring Boot service with the cleanest intentions: strict hexagonal or layered architecture, pristine package boundaries, isolated database transactions, and decoupled modules.

Yet, 12 months and 200 pull requests later, reality sets in:
- A `@RestController` directly invokes a Spring Data `@Repository` to bypass boilerplate "just this once."
- A circular dependency (`Class A -> Class B -> Class A`) quietly entangles two domain packages.
- A `@Transactional` method invokes a remote third-party REST API, silently holding open a database connection and starving your **HikariCP connection pool** during traffic spikes.
- A developer uses `this.internalMethod()` inside a `@Service`, inadvertently bypassing Spring's CGLIB proxy and silently breaking `@Transactional`, `@Async`, or `@Cacheable`.
- An engineer adopts **Java 21 Virtual Threads (Project Loom)**, but legacy `synchronized` blocks begin pinning underlying OS carrier threads, degrading throughput.

Most teams assume their CI/CD tools protect them. But the truth is:

* **Syntax linters (Checkstyle, PMD):** only inspect text/syntax; they don’t understand bytecode, type hierarchies, or Spring bean scopes.
* **Bytecode bug finders (SpotBugs):** inspect isolated instructions, completely unaware of layered architecture or Spring proxies.
* **ArchUnit:** is fantastic, but requires writing, maintaining, and executing hundreds of lines of imperative JUnit test code across dozens of repositories.
* **Enterprise platforms (SonarQube):** require hosting dedicated servers, databases, managing licenses, and still miss runtime Spring anti-patterns.

To solve this gap, we built and released **Java Application Quality Guard (`architecture-guard-maven-plugin`)** — now available on **Maven Central**.

---

## 2. What is Architecture Guard?

**Architecture Guard** is a static architecture analysis tool and zero-infrastructure Maven plugin engineered specifically for **Java 21+** and **Spring Boot 3+ / 6+**.

It couples **ASM Bytecode Analysis** (for fast, deep class dependency graphs) with **JavaParser AST Inspection** (for rich source-level semantics) into a single plugin that runs directly inside your local build:

```bash
mvn clean verify
```

No external servers. No background daemons. No databases. 100% offline, local, and reproducible in any CI/CD runner.

---

## 3. Killer Features That Set Architecture Guard Apart

### ⚡ 30 Out-of-the-Box Architectural & Concurrency Rules (AG001 – AG030)
Instead of forcing you to write boilerplate tests from scratch, Architecture Guard comes pre-loaded with an enterprise catalog covering:
* **Layer Boundary Enforcement:** Enforces `Controller ➔ Service ➔ Repository` flows and flags inverted dependencies or layer skips.
* **High-Performance Cycle Detection:** Implements Tarjan’s $O(V+E)$ Strongly Connected Components (SCC) and DFS to catch circular dependencies with canonical path reporting.
* **Spring Boot Anti-Pattern Detection:** Flags field injection (`@Autowired` on fields), missing `@Transactional` across multi-repository mutations, direct `@Entity` exposure on REST endpoints, and proxy self-invocations.

### 🧵 Modern Concurrency & Java 21 Virtual Thread Protection
As teams migrate to Java 21+, concurrency pitfalls change:
* **Virtual Thread Pinning (AG027):** Detects `synchronized` methods/blocks that pin OS carrier threads in Project Loom, recommending `ReentrantLock` instead.
* **ThreadLocal Memory Leaks (AG028):** Flags `ThreadLocal.set()` calls that lack guaranteed `remove()` execution in an enclosing `finally` block — preventing worker pool cross-request data leaks.
* **Unbounded `@Async` Thread Pools (AG029):** Flags `@Async` annotations missing explicit `ThreadPoolTaskExecutor` qualifiers to prevent native OS thread exhaustion.

### 🛡️ Production Resilience & Connection Pool Safeguards
* **Remote I/O inside `@Transactional` (AG026):** Forbids HTTP client, gRPC, and cloud SDK calls inside transaction boundaries. This single rule prevents cascading database outages caused by third-party latency holding open DB connections.
* **Missing HTTP Client Timeouts (AG030):** Flags unconfigured `RestTemplate`, `RestClient`, and `HttpClient` instances lacking explicit connect and read timeouts.

### 🎯 Technical Debt Baselining (Adopt Without Breaking Legacy Builds)
Turning on strict architectural rules on a 5-year-old codebase usually results in hundreds of violations and instant developer backlash.

Architecture Guard features **Differential Baselining**:
```bash
mvn architecture-guard:baseline
```
It snapshots existing technical debt. On subsequent builds, violations are classified as:
* `EXISTING`: Tracked without failing the build.
* `NEW`: Fails CI only if new code violates architectural standards.
* `RESOLVED`: Automatically removed as legacy code gets refactored.

### 📊 Multi-Format Developer Experience & 1-Click IDE Deep Linking
Architecture analysis shouldn't be buried in thousands of lines of terminal logs. Architecture Guard generates:
* **Standalone Offline HTML Dashboards:** Filterable reports with visual distribution charts and **1-click IDE deep linking** (`idea://open?file=...` and `vscode://file/...`) to jump straight to the offending line of code.
* **OASIS SARIF 2.1.0 Reports:** Upload directly to GitHub Actions to render native inline annotations on Pull Requests.
* **ANSI Terminal Summaries:** Clean, color-coded console summaries during local builds.

### 🚀 Blazing Enterprise Performance & Centralized Concurrency
Enterprise builds cannot tolerate static analysis delays. Architecture Guard features a centralized, isolated multithreading engine:
* **Concurrent Dual Engine:** Executes ArchUnit bytecode ingestion and JavaParser AST scanning in parallel worker threads, analyzing thousands of classes in **under 2.5 seconds**.
* **Isolated Thread Management:** Avoids polluting the shared JVM `commonPool`, using dedicated daemon threads (`archguard-worker-N`) with zero risk of worker starvation in Maven CI runners.
* **Deterministic & Scalable:** Deterministic file sorting and flexible thread sizing (`<threads>N</threads>` or `-Darchguard.threads=N`) ensure identical, predictable results across local machines and high-core CI servers.

---

## 4. Quick Start: Integrating into Your `pom.xml`

You can add Architecture Guard to any Maven project in less than 60 seconds:

```xml
<build>
    <plugins>
        <plugin>
            <groupId>io.github.javatechgroup</groupId>
            <artifactId>architecture-guard-maven-plugin</artifactId>
            <version>1.0.4</version>
            <executions>
                <execution>
                    <goals>
                        <goal>check</goal>
                    </goals>
                </execution>
            </executions>
            <configuration>
                <!-- Fail on ERROR, or set to NONE / WARNING for an initial non-blocking audit -->
                <failOnSeverity>ERROR</failOnSeverity>
            </configuration>
        </plugin>
    </plugins>
</build>
```

### In-Code Auditable Suppressions
When you have a legitimate architectural exception, you don't need messy configuration overrides. Suppress violations directly at the class or method level:

```java
@SuppressWarnings("archguard:AG001")
public class LegacyReportController {
    // Audited exception: directly calls repository for batch export
}
```
*Suppressions bypass build failures while remaining fully visible and auditable in the generated HTML dashboard.*

---

## 5. Getting Started & Essential Links

* 📦 **Maven Central Repository:**  
  https://central.sonatype.com/artifact/io.github.javatechgroup/architecture-guard-maven-plugin/versions

* 📖 **Complete Documentation, 30-Rule Catalog & User Manual:**  
  https://github.com/javatechgroup/archGaurdDocs

---

## Final Thoughts: Treat Architecture as Code

Architectural decay doesn't happen overnight; it happens one small shortcut at a time. By embedding automated, zero-infrastructure guardrails directly into your build lifecycle, you ensure your architecture remains clean, maintainable, and resilient from the first commit to year five in production.
