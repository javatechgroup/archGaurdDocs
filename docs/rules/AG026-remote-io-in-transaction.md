# AG026: Remote I/O in @Transactional Boundary

| Attribute | Details |
| :--- | :--- |
| **Rule ID** | `AG026` |
| **Name** | Remote I/O in @Transactional Boundary |
| **Default Severity** | `ERROR` |
| **Category** | Database Resilience |
| **Analysis Engine** | AST Source Analysis (JavaParser) |
| **Core Classes** | `AG026RemoteIoInTransactionRule, PerformanceInspector` |

---

## Executive Summary & Architectural Rationale

Holding a database transaction open while executing external network I/O is a leading cause of catastrophic production database outages:
1. **HikariCP Connection Pool Exhaustion:** When entering a `@Transactional` method, Spring acquires a physical database connection from HikariCP. If the method makes an external HTTP or payment API call that takes 3 to 10 seconds (or hangs on network timeout), that physical database connection remains completely idle and locked! Under moderate load, all HikariCP connections become exhausted in seconds, starving all database queries across the entire microservice.
2. **Database Row Lock Escalation:** Uncommitted database row locks (e.g. `SELECT FOR UPDATE`) remain held across network roundtrips, cascading into deadlocks across concurrent transactions.
3. **Non-Rollbackable Side Effects:** Network calls (sending emails, charging credit cards) cannot be rolled back if the subsequent database commit fails.

---

## How It Is Implemented

AST analysis inspects methods in classes where `@Transactional` is declared at the class or method level:
1. Checks if class or method is annotated with `@Transactional`.
2. Inspects all `MethodCallExpr` nodes inside the method.
3. **Remote Client Detection:**
   - Detects calls on injected client fields: `RestTemplate`, `RestClient`, `WebClient`, `HttpClient`, `JavaMailSender`, AWS S3/DynamoDB/SQS, `RabbitTemplate`, `KafkaTemplate`, Feign clients.
   - Detects remote network method names: `getForObject`, `postForObject`, `exchange`, `retrieve`, `send`, `sendMail`, `sendEmail`.
4. Emits `RemoteIoInTransactionFinding` with client category and exact line number.

---

### False Positive (FP) Handling & Mitigations
1. **Methods Without `@Transactional`:**
   - Network calls made in regular, non-transactional service methods do not hold database connections and are completely compliant.
2. **Local Method Invocations:**
   - Calls to local helper methods or domain calculations inside transactions do not trigger remote I/O alerts.

---

### False Negative (FN) Handling & Defense-in-Depth
1. **Class-Level `@Transactional`:**
   - Evaluates methods in classes annotated with class-level `@Transactional`, ensuring methods that inherit transaction status without explicit method annotations are captured.
2. **Broad Client Inventory:**
   - Covers HTTP clients, SMTP email senders, message brokers (Kafka/RabbitMQ), and Cloud SDKs.

---

## Code Examples

### Non-Compliant (Violating Anti-Pattern)

```java
// VIOLATION: External payment HTTP call holds physical DB connection open
@Service
public class OrderService {

    private final RestClient paymentRestClient;
    private final OrderRepository orderRepository;

    @Transactional // Database connection locked from start to finish!
    public void checkout(Order order) {
        orderRepository.save(order);

        // Slow network call (holds HikariCP connection idle for seconds!)
        paymentRestClient.post()
                .uri("/charge")
                .body(order.getAmount())
                .retrieve()
                .toBodilessEntity();
    }
}
```

### Compliant (Refactored Best Practice)

```java
// COMPLIANT: Network I/O decoupled from transactional persistence
@Service
@RequiredArgsConstructor
public class OrderOrchestrator {

    private final RestClient paymentRestClient;
    private final OrderPersistenceService orderPersistenceService;

    public void checkout(Order order) {
        // Step 1: Make network call OUTSIDE of transaction
        paymentRestClient.post()
                .uri("/charge")
                .body(order.getAmount())
                .retrieve()
                .toBodilessEntity();

        // Step 2: Atomic database write in dedicated short transaction
        orderPersistenceService.saveCompletedOrder(order);
    }
}
```

### Legitimate Edge Cases Handled Without Alerting

```java
// COMPLIANT: Non-transactional service method calling remote REST client
@Service
public class CurrencyService {
    private final RestTemplate restTemplate;

    // No @Transactional: database connection is never acquired
    public double getRate(String from, String to) {
        return restTemplate.getForObject("/rate?from=" + from, Double.class);
    }
}
```

---

## Configuration & Tuning

In `archguard.yml`, you can customize rule severity, exclude specific classes, or adjust thresholds:

```yaml
rules:
  AG026:
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
@SuppressArchitecture("AG026")
public void myMethod() {
    // ...
}

// Or using standard Java @SuppressWarnings
@SuppressWarnings("archguard:AG026")
public void myMethod() {
    // ...
}
```
