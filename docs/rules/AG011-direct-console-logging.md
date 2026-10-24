# AG011: Direct Console Logging

| Attribute | Details |
| :--- | :--- |
| **Rule ID** | `AG011` |
| **Name** | Direct Console Logging |
| **Default Severity** | `WARNING` |
| **Category** | Production Observability |
| **Analysis Engine** | AST Source Analysis (JavaParser) |
| **Core Classes** | `AG011DirectConsoleLoggingRule, ConsoleLoggingInspector` |

---

## Executive Summary & Architectural Rationale

Direct console writing in cloud and production microservices creates severe reliability and observability issues:
1. **Thread Blocking & Bottlenecks:** `System.out.println` synchronizes on a shared `PrintStream` lock internally. Under high concurrent request volume, all worker threads stall waiting for the console monitor lock.
2. **Bypasses Structured Observability:** Console prints bypass JSON formatters, log levels (INFO, WARN, ERROR), MDC correlation IDs, and file appender rotation policies.
3. **Log Ingestion Loss:** Log forwarders (Datadog, Splunk, Elastic) cannot parse unstructured stdout text, making tracing user transactions across microservices impossible.

---

## How It Is Implemented

AST inspection evaluates all `MethodCallExpr` nodes:
1. Matches method calls where scope is `System.out` or `System.err` and method is `print`, `println`, or `printf`.
2. Matches `e.printStackTrace()` with zero arguments.
3. Records `DirectConsoleLogFinding` with exact line number, enclosing method, and source code snippet.

---

### False Positive (FP) Handling & Mitigations
1. **SLF4J, Log4j, and Custom Loggers:**
   - Standard logger calls (`log.info(...)`, `logger.error(...)`) have non-System scopes and are never flagged.
2. **Main Application & CLI Entry Points (`public static void main`):**
   - Methods matching `public static void main(String[] args)` (such as standalone CLI tools, test runners, benchmark harnesses, or scratch runner scripts) are automatically exempted because direct stdout/stderr is standard and expected for standalone terminal execution.
3. **CLI & Console Commands:**
   - Dedicated Spring CLI runners (e.g. Spring Boot `CommandLineRunner` or Picocli) can suppress this rule using `@SuppressArchitecture("AG011")` if console output is the intended user interface.

---

### False Negative (FN) Handling & Defense-in-Depth
1. **Direct `System.err` Streams:**
   - Flags `System.err.println` in addition to `System.out`.
2. **Raw `printStackTrace()` in Catch Clauses:**
   - Catches developers calling `e.printStackTrace()` in catch blocks instead of `log.error("...", e)`.

---

## Code Examples

### Non-Compliant (Violating Anti-Pattern)

```java
// VIOLATION: Console output and raw printStackTrace
@RestController
public class PaymentController {

    @PostMapping("/pay")
    public void pay() {
        System.out.println("Processing payment..."); // Thread lock contention!
        try {
            // ...
        } catch (Exception e) {
            e.printStackTrace(); // Loss of MDC and structured log format!
        }
    }
}
```

### Compliant (Refactored Best Practice)

```java
// COMPLIANT: Structured logging via SLF4J
@Slf4j
@RestController
public class PaymentController {

    @PostMapping("/pay")
    public void pay() {
        log.info("Processing payment request");
        try {
            // ...
        } catch (Exception e) {
            log.error("Payment processing failed: {}", e.getMessage(), e);
        }
    }
}
```

### Legitimate Edge Cases Handled Without Alerting

```java
// COMPLIANT: Standalone CLI entry point or scratch runner
public class MigrationRunner {
    public static void main(String[] args) {
        System.out.println("Migration completed successfully."); // Permitted in main()
    }
}

// COMPLIANT: CLI runner with intentional architectural suppression
@Component
public class DatabaseMigrationCli implements CommandLineRunner {

    @Override
    @SuppressArchitecture("AG011") // Explicit user-facing terminal interface
    public void run(String... args) {
        System.out.println("Starting database migration CLI...");
    }
}
```

---

## Configuration & Tuning

In `archguard.yml`, you can customize rule severity, exclude specific classes, or adjust thresholds:

```yaml
rules:
  AG011:
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
@SuppressArchitecture("AG011")
public void myMethod() {
    // ...
}

// Or using standard Java @SuppressWarnings
@SuppressWarnings("archguard:AG011")
public void myMethod() {
    // ...
}
```
