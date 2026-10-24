# AG015: Missing @Valid on REST @RequestBody

| Attribute | Details |
| :--- | :--- |
| **Rule ID** | `AG015` |
| **Name** | Missing @Valid on REST @RequestBody |
| **Default Severity** | `WARNING` |
| **Category** | API Resilience |
| **Analysis Engine** | AST Source Analysis (JavaParser) |
| **Core Classes** | `AG015MissingValidationRule, RestApiInspector, AstUtils` |

---

## Executive Summary & Architectural Rationale

Unvalidated HTTP request payloads pose reliability and security hazards:
1. **Garbage Input Propagation:** Null values, blank strings, negative numbers, or invalid email formats bypass the presentation layer and trigger unexpected `NullPointerException` or database constraint violations in deep service layers.
2. **Defensive Coding Overhead:** Forces services to duplicate boilerplate null/empty checks on every method.
3. **Inconsistent Error Responses:** Without Bean Validation, validation failures return unhelpful 500 Internal Server Errors instead of standard RFC 7807 400 Bad Request error envelopes.

---

## How It Is Implemented

AST analysis inspects method parameters in `@RestController` classes:
1. Identifies parameters annotated with `@RequestBody`.
2. Checks if payload type is non-validatable via `AstUtils.isNonValidatablePayload()`.
3. Verifies if `@Valid` (Jakarta Validation) or `@Validated` (Spring) is present on the parameter.
4. If missing, records `MissingValidationFinding` with line number and parameter details.

---

### False Positive (FP) Handling & Mitigations
1. **Raw & Streaming Payloads:**
   - Byte arrays (`byte[]`, `Byte[]`, `ByteBuffer`), input streams (`InputStream`, `Resource`, `InputStreamResource`), and file uploads (`MultipartFile`, `Part`) cannot be validated by Hibernate Validator and are automatically exempted in `AstUtils.isNonValidatablePayload()`.
2. **Raw Strings & JSON Nodes:**
   - Raw `String`, `CharSequence`, `JsonNode`, `ObjectNode`, and generic `Map<String, Object>` payloads are exempted.
3. **Accepts both `@Valid` and `@Validated`:**
   - Recognizes both standard Jakarta `@Valid` and Spring's group-validation `@Validated`.

---

### False Negative (FN) Handling & Defense-in-Depth
1. **Fully Qualified Annotations:**
   - Checks `jakarta.validation.Valid`, `javax.validation.Valid`, and `org.springframework.validation.annotation.Validated`.
2. **Custom DTO Records:**
   - Enforces validation on Java 21 Records and custom request classes.

---

## Code Examples

### Non-Compliant (Violating Anti-Pattern)

```java
// VIOLATION: @RequestBody parameter without validation
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @PostMapping
    public ResponseEntity<Void> createOrder(@RequestBody CreateOrderRequest request) { // Missing @Valid!
        orderService.create(request);
        return ResponseEntity.ok().build();
    }
}
```

### Compliant (Refactored Best Practice)

```java
// COMPLIANT: @Valid triggers automated Jakarta Bean Validation
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @PostMapping
    public ResponseEntity<Void> createOrder(@Valid @RequestBody CreateOrderRequest request) {
        orderService.create(request);
        return ResponseEntity.ok().build();
    }
}
```

### Legitimate Edge Cases Handled Without Alerting

```java
// COMPLIANT: Raw byte stream payload does not require @Valid
@RestController
public class FileUploadController {
    @PostMapping(value = "/upload", consumes = "application/octet-stream")
    public void upload(@RequestBody byte[] rawBytes) { // Non-validatable payload: permitted
        // ...
    }
}
```

---

## Configuration & Tuning

In `archguard.yml`, you can customize rule severity, exclude specific classes, or adjust thresholds:

```yaml
rules:
  AG015:
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
@SuppressArchitecture("AG015")
public void myMethod() {
    // ...
}

// Or using standard Java @SuppressWarnings
@SuppressWarnings("archguard:AG015")
public void myMethod() {
    // ...
}
```
