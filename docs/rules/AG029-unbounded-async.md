# AG029: Unbounded @Async Thread Pool

| Attribute | Details |
| :--- | :--- |
| **Rule ID** | `AG029` |
| **Name** | Unbounded @Async Thread Pool |
| **Default Severity** | `WARNING` |
| **Category** | Threading & Resource Safety |
| **Analysis Engine** | AST Source Analysis (Two-Pass Post-Processing) |
| **Core Classes** | `AG029UnboundedAsyncRule, ConcurrencyInspector` |

---

## Executive Summary & Architectural Rationale

In Spring Framework, using `@Async` without an explicit bounded thread pool is a high-risk operational trap:
1. **Unbounded Thread Creation (`SimpleAsyncTaskExecutor`):** By default, Spring's `@Async` falls back to `SimpleAsyncTaskExecutor`, which does **not** reuse threads. It spawns a brand new native OS thread for *every single asynchronous task invocation*.
2. **Native Thread Exhaustion (`OutOfMemoryError: unable to create native thread`):** Under a surge of requests, hundreds or thousands of OS threads are created, crashing the JVM and the host operating system.
3. **Context Switching Overhead:** Massive thread counts cause devastating CPU context-switching thrashing.

---

## How It Is Implemented

Uses a **Two-Pass AST Architecture** to resolve global configuration and bean qualifiers:
1. **Pass 1 (Configuration & Bean Discovery):**
   - Checks if any class implements `AsyncConfigurer` or `AsyncConfigurerSupport` (which provides a global custom executor via `getAsyncExecutor()`).
   - Scans `@Bean` methods returning `Executor`, `TaskExecutor`, `ThreadPoolTaskExecutor`, or named `taskExecutor` / `applicationTaskExecutor` / `defaultTaskExecutor`.
   - Collects `@Async` candidate methods:
     - Unparameterized `@Async` (candidates for global pool check)
     - Parameterized `@Async("poolName")` (candidates for qualifier check)
2. **Pass 2 (`postProcess`):**
   - For unparameterized `@Async`: if project has `AsyncConfigurer` or standard default task executor beans, marks compliant; otherwise records violation.
   - For parameterized `@Async("poolName")`: verifies that `"poolName"` matches an existing declared `@Bean` in the project. If qualifier points to a non-existent bean, records violation.

---

### False Positive (FP) Handling & Mitigations
1. **Projects Implementing `AsyncConfigurer`:**
   - When a project implements `AsyncConfigurer` (e.g. `public class AsyncConfig implements AsyncConfigurer`), unparameterized `@Async` automatically delegates to `getAsyncExecutor()`. The inspector detects this and does **not** flag unparameterized `@Async`.
2. **Standard Default Beans:**
   - Declaring a `@Bean(name = "taskExecutor")` or `@Bean public ThreadPoolTaskExecutor defaultTaskExecutor()` satisfies unparameterized `@Async`.
3. **Qualified Executors:**
   - Methods specifying `@Async("customExecutor")` where `@Bean("customExecutor")` exists in configuration are compliant.

---

### False Negative (FN) Handling & Defense-in-Depth
1. **Non-Existent Executor Qualifiers:**
   - If a developer types `@Async("ordersExecutor")` but the bean was renamed or deleted in configuration, Pass 2 verifies bean existence and flags the broken qualifier.
2. **Two-Pass Cross-File Resolution:**
   - Configuration beans declared in `AsyncConfiguration.java` are recognized even when analyzing service classes parsed earlier.

---

## Code Examples

### Non-Compliant (Violating Anti-Pattern)

```java
// VIOLATION: Unbounded @Async without configured thread pool
@Service
public class EmailService {

    @Async // Defaults to SimpleAsyncTaskExecutor: creates new OS thread per email!
    public void sendEmail(String to, String body) {
        // ...
    }
}
```

### Compliant (Refactored Best Practice)

```java
// COMPLIANT: Parameterized @Async pointing to bounded ThreadPoolTaskExecutor
@Service
public class EmailService {

    @Async("emailTaskExecutor") // Bounded, managed thread pool
    public void sendEmail(String to, String body) {
        // ...
    }
}

// In Configuration: Explicit bounded thread pool
@Configuration
@EnableAsync
public class AsyncConfig {
    @Bean("emailTaskExecutor")
    public ThreadPoolTaskExecutor emailTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(500);
        executor.setThreadNamePrefix("email-exec-");
        executor.initialize();
        return executor;
    }
}
```

### Legitimate Edge Cases Handled Without Alerting

```java
// COMPLIANT: Global AsyncConfigurer implementation
@Configuration
@EnableAsync
public class GlobalAsyncConfig implements AsyncConfigurer {
    @Override
    public Executor getAsyncExecutor() {
        return new ThreadPoolTaskExecutor(); // Governs all unparameterized @Async
    }
}
```

---

## Configuration & Tuning

In `archguard.yml`, you can customize rule severity, exclude specific classes, or adjust thresholds:

```yaml
rules:
  AG029:
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
@SuppressArchitecture("AG029")
public void myMethod() {
    // ...
}

// Or using standard Java @SuppressWarnings
@SuppressWarnings("archguard:AG029")
public void myMethod() {
    // ...
}
```
