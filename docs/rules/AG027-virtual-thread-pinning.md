# AG027: Java 21 Virtual Thread Pinning

| Attribute | Details |
| :--- | :--- |
| **Rule ID** | `AG027` |
| **Name** | Java 21 Virtual Thread Pinning |
| **Default Severity** | `WARNING` |
| **Category** | Modern Java Concurrency |
| **Analysis Engine** | AST Source Analysis + Target Java Version Gate |
| **Core Classes** | `AG027VirtualThreadPinningRule, ConcurrencyInspector` |

---

## Executive Summary & Architectural Rationale

In Java 21 Virtual Threads (Project Loom), millions of lightweight virtual threads are multiplexed across a small pool of OS carrier threads:
1. **Carrier Thread Pinning:** In JDK 21 to 23, when a virtual thread acquires a monitor lock (`synchronized` method or `synchronized (obj)` block) and performs blocking I/O (database query, socket read), it cannot unmount from its underlying OS carrier thread. The carrier thread becomes "pinned" and blocked.
2. **Severe Throughput Collapse:** When all available carrier threads (defaulting to the number of CPU cores) are pinned, the entire virtual thread scheduler stalls, collapsing application throughput.
3. **Deadlock Hazards on Unreleased Locks:** When replacing `synchronized` with `ReentrantLock`, acquiring `lock.lock()` without releasing it inside a `finally` block creates permanent application deadlocks.

---

## How It Is Implemented

Evaluates synchronized constructs and concurrency locks:
1. **JDK Version Awareness:**
   - Reads `context.getTargetJavaVersion()`.
   - Under JDK 24+ (JEP 491: *Synchronize Virtual Threads without Pinning*), monitor locks no longer pin carrier threads. AG027 **automatically deactivates** monitor lock warnings on Java 24+.
2. **Monitor Lock Detection (JDK 21-23):**
   - Inspects methods for `method.isSynchronized()` and `SynchronizedStmt`.
3. **Unreleased Lock Inspection (All JDKs):**
   - Inspects `lock.lock()` or `lock.lockInterruptibly()`.
   - Verifies whether `isSafelyReleasedInFinally()` confirms a corresponding `lock.unlock()` inside an enclosing `try ... finally` block.
   - Flags missing `finally { unlock(); }` as `ERROR` severity on all Java versions!

---

### False Positive (FP) Handling & Mitigations
1. **Automatic Deactivation for Java 24+:**
   - When projects specify `<java.version>24</java.version>`, monitor pinning warnings are suppressed automatically, recognizing JEP 491.
2. **Correctly Paired `ReentrantLock`:**
   - Acquiring a lock paired with `try { ... } finally { lock.unlock(); }` is compliant and not flagged.
3. **Thread-Safe Concurrent Collections:**
   - Using `ConcurrentHashMap` or `AtomicInteger` does not use monitor locks and is fully compliant.

---

### False Negative (FN) Handling & Defense-in-Depth
1. **Unreleased Lock Deadlock Detection:**
   - Detects developers who refactored away from `synchronized` to `ReentrantLock` but forgot the `finally { lock.unlock(); }` block, preventing fatal concurrency deadlocks.
2. **Scope-Aware Unlock Matching:**
   - Checks that the lock variable name matches the unlock variable name.

---

## Code Examples

### Non-Compliant (Violating Anti-Pattern)

```java
// VIOLATION: Synchronized method pins carrier thread during I/O on Java 21
@Service
public class ReportService {

    // Synchronized keyword pins carrier thread in Project Loom!
    public synchronized Report generateReport(Long id) {
        return databaseClient.fetchLargeReport(id); // Blocking I/O while pinned!
    }
}

// FATAL VIOLATION: ReentrantLock acquired without finally unlock!
public void updateState() {
    lock.lock(); // If next line throws, lock is NEVER released! Permanent deadlock.
    processState();
    lock.unlock();
}
```

### Compliant (Refactored Best Practice)

```java
// COMPLIANT: ReentrantLock properly released in finally block
@Service
public class ReportService {

    private final ReentrantLock lock = new ReentrantLock();

    public Report generateReport(Long id) {
        lock.lock();
        try {
            return databaseClient.fetchLargeReport(id); // Virtual thread unmounts safely
        } finally {
            lock.unlock(); // Guaranteed lock release
        }
    }
}
```

### Legitimate Edge Cases Handled Without Alerting

```java
// COMPLIANT: JDK 24+ target automatically suppresses synchronized warnings
// With target Java 24, JEP 491 resolves monitor pinning in JVM runtime
public synchronized void syncMethod() {
    // Compliant on Java 24+
}
```

---

## Configuration & Tuning

In `archguard.yml`, you can customize rule severity, exclude specific classes, or adjust thresholds:

```yaml
rules:
  AG027:
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
@SuppressArchitecture("AG027")
public void myMethod() {
    // ...
}

// Or using standard Java @SuppressWarnings
@SuppressWarnings("archguard:AG027")
public void myMethod() {
    // ...
}
```
