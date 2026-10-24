# Architecture Guard Rules Reference

This document details all built-in rules supported by Java Application Quality Guard.

> [!NOTE]
> **Detailed Rule Implementation & False Positive/Negative Guides:**
> For an in-depth technical analysis of each rule's implementation, AST/bytecode detection logic, and false positive / false negative handling, see the dedicated [Rules Catalog](./README.md) and individual rule specifications in this directory.

> [!TIP]
> **Concurrent High-Throughput Execution:**
> All 33 rules are evaluated concurrently across the bytecode engine (AG001–AG006) and AST source engine (AG007–AG033) using the centralized [`ArchGuardTaskExecutor`](../technical/architecture.md#81-centralized-concurrency-architecture). Thread sizing is customizable via `<threads>N</threads>` in POM, `analysis.threads` in `archguard.yml`, or `-Darchguard.threads=N` via CLI.

---

### AG001: Controller to Repository Direct Dependency
- **Default Severity**: `ERROR`
- **Category**: Layer Boundary Enforcement
- **Description**: Controllers must not have direct references to Repositories.
- **Why it matters**: Direct access from web controllers to repositories bypasses transaction boundaries, business validations, and auditing logic typically placed in the Service layer.
- **Compliant Pattern**: `Controller -> Service -> Repository`
- **Violating Pattern**: `Controller -> Repository`
- **Remediation**: Create or inject a Service class to encapsulate the database operation.

---

### AG002: Controller to Controller Direct Dependency
- **Default Severity**: `ERROR`
- **Category**: Layer Boundary Enforcement
- **Description**: Controllers must not directly depend on or call other controllers.
- **Why it matters**: Web controllers are entry points with request handling concerns; cross-calling them couples endpoint logic tightly.
- **Remediation**: Extract common logic to a shared Service and inject that service into both controllers.

---

### AG003: Service to Controller Inverted Dependency
- **Default Severity**: `ERROR`
- **Category**: Inverted Layer Hierarchy
- **Description**: Services must not depend on web controllers.
- **Why it matters**: Services represent the domain and business layer. Coupling them to the presentation/web layer prevents reuse across CLI, message consumers, or other presentation adapters.
- **Remediation**: Remove controller references from service classes. Pass necessary DTOs or arguments into service methods.

---

### AG004: Repository to Service Inverted Dependency
- **Default Severity**: `ERROR`
- **Category**: Inverted Layer Hierarchy
- **Description**: Repositories must not depend on services.
- **Why it matters**: Data persistence components must remain at the bottom of the dependency hierarchy.
- **Remediation**: Move business coordination logic into the service layer.

---

### AG005: Class-Level Cyclic Dependency
- **Default Severity**: `ERROR`
- **Category**: Coupling & Maintainability
- **Description**: Detects circular dependency chains (`ClassA -> ClassB -> ClassA`).
- **Why it matters**: Cyclic dependencies prevent isolated unit testing, complicate initialization order, and tightly couple classes together.
- **Algorithm**: Tarjan's Strongly Connected Components (SCC) and DFS cycle tracing ($O(V+E)$).
- **Remediation**: Break the cycle by extracting shared responsibilities to a new component, using interfaces (Dependency Inversion), or leveraging event-driven messaging.

---

### AG006: Package Boundary Validation
- **Default Severity**: `WARNING`
- **Category**: Custom Architectural Boundaries
- **Description**: Ensures that classes in package A can only access packages explicitly declared in `packageBoundaries` configuration.
- **Remediation**: Refactor package references or adjust permitted package boundaries in `archguard.yml`.

---

### AG007: Prohibited Field Injection
- **Default Severity**: `WARNING`
- **Category**: Clean Code & Testability
- **Description**: Flags `@Autowired`, `@Inject`, or `@Resource` annotations placed directly on private fields.
- **Why it matters**: Field injection hides dependencies, prevents immutability (`final` fields), and makes unit testing difficult without reflection.
- **Remediation**: Use constructor injection or Lombok's `@RequiredArgsConstructor`.

---

### AG008: Swallowed Exceptions
- **Default Severity**: `WARNING`
- **Category**: Error Handling & Reliability
- **Description**: Detects empty `catch` blocks or caught exceptions that are neither logged nor rethrown.
- **Why it matters**: Swallowed exceptions silently hide production failures, corrupt transactional state, and turn debugging into guesswork.
- **Remediation**: Log the exception with a logger (`log.error("...", e)`), rethrow it wrapped in a domain exception, or restore interrupted status with `Thread.currentThread().interrupt()` when catching `InterruptedException`.

---

### AG009: Missing @Transactional on Multi-Repository Writes
- **Default Severity**: `WARNING`
- **Category**: Data Integrity & Transactions
- **Description**: Detects multiple mutating repository calls in a public service method lacking `@Transactional`.
- **Why it matters**: Without an explicit transaction boundary, partial database writes will commit if a subsequent operation throws an exception, corrupting database consistency. Read-only query lookups (`find*`, `get*`, `count*`, `exists*`) are safe and do not trigger this alert.
- **Remediation**: Add `@Transactional` to the service method or wrap writes in a `TransactionTemplate`.

---

### AG010: Unsafe Mutable Static State
- **Default Severity**: `ERROR`
- **Category**: Concurrency & Thread Safety
- **Description**: Detects static mutable fields or collections in singleton Spring beans (`@Service`, `@Component`, `@RestController`).
- **Why it matters**: Static mutable state is shared across all concurrent worker threads, leading to race conditions and data corruption.
- **Remediation**: Make the field `final` or replace mutable collections with immutable/concurrent structures.

---

### AG011: Direct Console Logging
- **Default Severity**: `WARNING`
- **Category**: Production Observability
- **Description**: Detects `System.out.println`, `System.err.println`, or `e.printStackTrace()`.
- **Why it matters**: Console statements bypass structured logging formats, log levels, correlation IDs, and file rotation.
- **Remediation**: Use an SLF4J logger: `log.info(...)`, `log.warn(...)`, or `log.error(...)`.

---

### AG012: Direct Entity Exposure in REST Controllers
- **Default Severity**: `ERROR`
- **Category**: Security & API Governance
- **Description**: Forbids `@RestController` endpoints from accepting or returning database `@Entity` / `@Document` classes directly. Standard Spring UI/Web structures (`Model`, `ModelMap`, `ModelAndView`) and non-entity DTOs are permitted.
- **Why it matters**: Exposing JPA entities causes mass-assignment vulnerabilities, accidental schema leaks, and `LazyInitializationException` crashes during JSON serialization.
- **Remediation**: Return and accept dedicated DTOs or Java 21 Records.

---

### AG013: Hardcoded Credentials & Secrets
- **Default Severity**: `ERROR`
- **Category**: Application Security
- **Description**: Detects hardcoded API keys, private keys, JWT tokens, AWS access keys, and passwords in source files. Header constants, URL paths, and configuration keys are appropriately excluded.
- **Why it matters**: Hardcoded credentials committed to version control lead to severe credential theft and unauthorized access.
- **Remediation**: Inject credentials from environment variables, Vault, or Spring Cloud Config via `@Value("${...}")`.

---

### AG014: JPA Eager Fetch Detection
- **Default Severity**: `WARNING`
- **Category**: Database Performance
- **Description**: Flags `FetchType.EAGER` on `@OneToMany` or `@ManyToMany` relationships.
- **Why it matters**: Eager fetching causes Cartesian product query generation and severe N+1 SQL performance bottlenecks.
- **Remediation**: Use `FetchType.LAZY` with targeted JPQL `JOIN FETCH` or `@EntityGraph`.

---

### AG015: Missing @Valid on REST @RequestBody
- **Default Severity**: `WARNING`
- **Category**: API Resilience
- **Description**: Enforces Bean Validation (`@Valid` or `@Validated`) on all `@RequestBody` parameters in mutation endpoints.
- **Why it matters**: Unvalidated payloads allow nulls, blank strings, or illegal formats to propagate into domain services.
- **Remediation**: Add `@Valid` or `@Validated` to the `@RequestBody` parameter.

---

### AG016: Unpaginated REST Collections
- **Default Severity**: `WARNING`
- **Category**: Memory Safety & API Scaling
- **Description**: Flags unpaginated `List<T>` or `Set<T>` return types from `@GetMapping` endpoints.
- **Why it matters**: Returning unbounded collections risks OutOfMemoryError and massive network transfer when tables grow.
- **Remediation**: Accept `Pageable` and return Spring Data's `Page<T>` or `Slice<T>`.

---

### AG017: Injected Dependency Bloat / God Class
- **Default Severity**: `WARNING`
- **Category**: Maintainability & Modularity
- **Description**: Flags classes with $\ge 7$ injected collaborators.
- **Why it matters**: High collaborator counts violate the Single Responsibility Principle and signal an unmanageable God Class.
- **Remediation**: Split responsibilities into smaller domain services or introduce Facade / Strategy patterns.

---

### AG018: High Cyclomatic Complexity
- **Default Severity**: `WARNING`
- **Category**: Maintainability & Code Quality
- **Description**: Flags methods with cyclomatic complexity $> 15$.
- **Why it matters**: Highly branching methods are difficult to comprehend, defect-prone, and hard to test thoroughly.
- **Remediation**: Extract cohesive sub-blocks into private helper methods or replace conditionals with polymorphic strategies.

---

### AG019: Algorithmic Performance Risk in Loops
- **Default Severity**: `WARNING`
- **Category**: Runtime Performance
- **Description**: Detects linear collection searches (`List.contains`, `List.indexOf`) or database queries inside loops. Distinguishes constant-time lookups (e.g., `Set.contains`) and string operations (`String.contains`) to avoid false positives.
- **Why it matters**: Linear searches create $O(n^2)$ CPU bottlenecks, and database queries in loops create N+1 roundtrips.
- **Remediation**: Use a `HashSet` for $O(1)$ lookups and batch database queries outside the loop (`findAllById(...)`).

---

### AG020: Code Readability & Nesting
- **Default Severity**: `WARNING`
- **Category**: Clean Code
- **Description**: Flags monster methods ($> 60$ lines), long parameter lists ($\ge 5$ parameters), and deep arrow anti-pattern nesting ($\ge 4$ levels). Sequential `else if` ladders are treated as flat control flow and not penalized.
- **Why it matters**: Deep nesting and oversized methods drastically impair code readability and maintenance.
- **Remediation**: Use Guard Clauses (return early) and encapsulate parameter lists into Java 21 Records / DTOs.

---

### AG021: Spring Proxy Self-Invocation
- **Default Severity**: `ERROR`
- **Category**: Spring Runtime Traps
- **Description**: Flags internal `this.method()` calls that silently bypass Spring AOP proxies (`@Transactional`, `@Async`, `@Cacheable`). Accurately checks target method parameter signatures so unannotated overloads are not misidentified.
- **Why it matters**: Spring's dynamic proxies only intercept calls through injected bean references; self-invocation calls execute as plain Java calls, silently ignoring transaction and caching behavior.
- **Remediation**: Extract the annotated method to a separate injected service component or use `TransactionTemplate`.

---

### AG022: Singleton Mutable Instance State
- **Default Severity**: `ERROR`
- **Category**: Concurrency & Thread Safety
- **Description**: Detects non-final mutable fields or non-thread-safe utilities (like `SimpleDateFormat`) in singleton Spring beans (`@Service`, `@Component`, `@RestController`). Request- or session-scoped beans (`@RequestScope`, `@SessionScope`) are exempted.
- **Why it matters**: Spring beans are singletons shared across concurrent HTTP request worker threads; mutable state causes race conditions and data leaks across users.
- **Remediation**: Make fields `final`, use thread-safe alternatives (`DateTimeFormatter`), or scope the bean appropriately.

---

### AG023: NullPointer Traps & Defensive Coding
- **Default Severity**: `WARNING`
- **Category**: Defensive Programming
- **Description**: Flags raw unguarded `Optional.get()`, methods returning `null` instead of empty collections, and inverted equality checks (`var.equals("LITERAL")`). Explicit presence guards (`opt.isPresent()`) and null checks (`var != null`) are respected.
- **Why it matters**: These patterns frequently cause `NoSuchElementException` and `NullPointerException`.
- **Remediation**: Use `Optional.orElseThrow()`, return `Collections.emptyList()`, and use Yoda comparison `"LITERAL".equals(var)`.

---

### AG024: AutoCloseable Resource Leaks
- **Default Severity**: `ERROR`
- **Category**: Resource Management
- **Description**: Flags I/O streams, readers, or JDBC connections created outside of try-with-resources.
- **Why it matters**: Unclosed resources leak operating system file descriptors and database connections, eventually crashing the application.
- **Remediation**: Wrap resource allocation in a `try (...)` block.

---

### AG025: Loop String Concatenation
- **Default Severity**: `WARNING`
- **Category**: Performance & Memory
- **Description**: Flags `+` or `+=` string concatenation inside loops (`for`, `while`). Numeric compound addition (`sum += val`) is recognized and excluded.
- **Why it matters**: Repeated string concatenation inside loops allocates intermediate String copies on every iteration, leading to $O(n^2)$ heap memory churn and GC pressure.
- **Remediation**: Use `StringBuilder.append(...)`.

---

### AG026: Remote I/O in @Transactional Boundary
- **Default Severity**: `ERROR`
- **Category**: Database Resilience
- **Description**: Prevents remote HTTP, REST, RPC, email, or cloud SDK calls inside `@Transactional` methods.
- **Why it matters**: Remote network calls hold the physical database connection from HikariCP locked idle, causing connection pool starvation and site-wide outages under load.
- **Remediation**: Execute remote calls outside the transaction using an Orchestrator pattern or `TransactionTemplate`.

---

### AG027: Java 21 Virtual Thread Pinning
- **Default Severity**: `WARNING`
- **Category**: Modern Java Concurrency
- **Description**: Detects `synchronized` methods or blocks that pin OS carrier threads in Project Loom (Virtual Threads). Automatically deactivated when target Java release is 24 or higher (JEP 491 resolves carrier thread pinning for monitor locks).
- **Why it matters**: In Java 21–23, pinning blocks the carrier thread during I/O, preventing the virtual thread scheduler from multiplexing tasks and crippling throughput.
- **Remediation**: Replace `synchronized` with `java.util.concurrent.locks.ReentrantLock` when targeting Java 21–23, or upgrade project target to Java 24+.

---

### AG028: ThreadLocal Memory Leak
- **Default Severity**: `ERROR`
- **Category**: Memory Safety & Concurrency
- **Description**: Flags `ThreadLocal.set()` calls that lack guaranteed `remove()` execution in an enclosing `finally` block or in standard paired lifecycle hooks (such as `HandlerInterceptor.afterCompletion`).
- **Why it matters**: Pooled web worker threads retain `ThreadLocal` references indefinitely, causing major memory leaks and cross-request security leaks.
- **Remediation**: Always execute `threadLocal.remove()` in a `finally` block, pair `preHandle` with `afterCompletion`, or adopt Java 21 `ScopedValue`.

---

### AG029: Unbounded @Async Thread Pool
- **Default Severity**: `WARNING`
- **Category**: Threading & Resource Safety
- **Description**: Flags `@Async` annotations missing a qualified bounded `TaskExecutor` bean name.
- **Why it matters**: Defaults to `SimpleAsyncTaskExecutor`, which creates a brand-new OS thread per invocation, leading to native thread exhaustion and crashes.
- **Remediation**: Provide a bounded executor bean name: `@Async("taskExecutor")`.

---

### AG030: Missing HTTP Client Timeout Configuration
- **Default Severity**: `ERROR`
- **Category**: Cloud Resilience & Microservice Reliability
- **Description**: Detects unconfigured `RestTemplate`, `RestClient`, and `HttpClient` instances lacking explicit connection and read timeouts.
- **Why it matters**: In Spring Boot and standard Java, default HTTP clients use infinite (0ms) timeouts. When a downstream microservice or payment gateway stalls, all Tomcat/Jetty worker threads calling that endpoint block indefinitely, causing complete thread starvation, connection pool exhaustion, and cascading site-wide outages.
- **Remediation**:
  - For `RestTemplate`: Use `RestTemplateBuilder.setConnectTimeout(...).setReadTimeout(...)` or configure a `ClientHttpRequestFactory`.
  - For `RestClient`: Use `RestClient.builder().requestFactory(...)` with explicit timeouts.
  - For `HttpClient`: Use `HttpClient.newBuilder().connectTimeout(Duration.ofSeconds(...)).build()`.

---

### AG031: Dynamic Query Concatenation Injection Risk
- **Default Severity**: `ERROR`
- **Category**: Security, Resilience & Safety
- **Standard Mapping**: OWASP Top 10 (A03:2021 — Injection), CWE-89
- **Description**: Detects unparameterized dynamic string concatenation in SQL or JPQL queries (`@Query`, `EntityManager`, `JdbcTemplate`).
- **Why it matters**: Direct string concatenation into database queries enables untrusted input to alter query logic, exposing sensitive data to SQL/JPQL injection attacks and causing database CPU churn via query recompilation.
- **Remediation**: Use named query parameters (`:paramName`) or positional parameters (`?1`) with PreparedStatement or Spring Data parameter binding.

---

### AG032: Broken Cryptographic Algorithm
- **Default Severity**: `ERROR`
- **Category**: Security, Resilience & Safety
- **Standard Mapping**: OWASP Top 10 (A02:2021 — Cryptographic Failures), CWE-327
- **Description**: Detects usage of obsolete hashing algorithms (`MD5`, `SHA-1`) or weak ciphers (`DES`, `RC4`, `ECB` mode).
- **Why it matters**: Deprecated hashing algorithms have proven collision vulnerabilities. Weak ciphers and ECB mode leak plaintext patterns, allowing attackers to decrypt sensitive stored or transmitted data.
- **Remediation**: Use collision-resistant hashing such as `SHA-256` or `SHA-512`, and authenticated encryption such as `AES/GCM/NoPadding`.

---

### AG033: Insecure CORS Configuration
- **Default Severity**: `ERROR`
- **Category**: Security, Resilience & Safety
- **Standard Mapping**: OWASP Top 10 (A05:2021 — Security Misconfiguration), CWE-942
- **Description**: Flags CORS policies permitting wildcard origins (`*`) while enabling credentials (`allowCredentials=true`).
- **Why it matters**: Combining wildcard origins with credential forwarding allows any arbitrary malicious website visited by a user to issue authenticated cross-origin requests and read sensitive responses with the user's cookies.
- **Remediation**: Specify explicit, trusted allowed origins (e.g. `https://app.example.com`) or remove `allowCredentials` when wildcard origins are needed.


