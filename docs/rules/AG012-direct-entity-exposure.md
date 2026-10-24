# AG012: Direct Entity Exposure in REST Controllers

| Attribute | Details |
| :--- | :--- |
| **Rule ID** | `AG012` |
| **Name** | Direct Entity Exposure in REST Controllers |
| **Default Severity** | `ERROR` |
| **Category** | Security & API Governance |
| **Analysis Engine** | AST Source Analysis (Two-Pass Post-Processing) |
| **Core Classes** | `AG012DirectEntityExposureRule, RestApiInspector, AstUtils` |

---

## Executive Summary & Architectural Rationale

Exposing database entities directly in REST controllers is a critical security vulnerability and operational hazard:
1. **Mass-Assignment / Over-Posting Vulnerability:** When JPA entities are accepted as `@RequestBody`, attackers can inject malicious fields (e.g. `isAdmin: true`, `role: SUPERUSER`, `balance: 999999`) that Spring automatically binds and persists directly into the database.
2. **Internal Schema Leakage:** Internal database details (password hashes, soft-delete flags, audit metadata) are inadvertently serialized into JSON responses.
3. **LazyInitializationException Crashes:** Serializing entities outside of an active persistence context causes Jackson to trigger uninitialized lazy proxies, throwing `LazyInitializationException` at runtime.

---

## How It Is Implemented

Uses a **Two-Pass AST Architecture** to resolve entity types regardless of compilation order:
1. **Pass 1 (Entity Discovery):**
   - Scans all parsed compilation units for class declarations with `@Entity`, `@Table`, `@Document`, or `@Embeddable`.
   - Records their simple and fully qualified names into `discoveredEntities`.
2. **Pass 1 (Controller Endpoint Inspection):**
   - In classes annotated with `@RestController`, `@Controller`, or `*Controller`, inspects all endpoint methods (`@GetMapping`, `@PostMapping`, etc.).
   - Unwraps wrappers: `ResponseEntity<T>`, `Optional<T>`, `List<T>`, `Set<T>`, `Collection<T>`, `Flux<T>`, `Mono<T>`.
   - If unwrapped type matches `discoveredEntities` or ends with `Entity`, records finding immediately.
   - Otherwise, queues candidate into `pendingExposures`.
3. **Pass 2 (`postProcess`):**
   - Re-evaluates all queued `pendingExposures` against the completed `discoveredEntities` universe, resolving forward-referenced entities.

---

### False Positive (FP) Handling & Mitigations
1. **Spring Web UI / MVC Model Objects:**
   - Framework classes like `Model`, `ModelMap`, `ExtendedModelMap`, `ConcurrentModel`, `ModelAndView`, `BindingResult`, `Errors`, `HttpServletRequest`, `HttpServletResponse`, `HttpSession`, `Principal`, `Authentication` are explicitly whitelisted and exempted in `AstUtils.NON_ENTITY_FRAMEWORK_TYPES`.
2. **ResponseEntity & HttpEntity Wrappers:**
   - Unwraps `ResponseEntity<UserDto>` to evaluate `UserDto`, avoiding false positives from the wrapper name itself.
3. **DTOs and Value Objects:**
   - Classes named `UserDto`, `UserResponse`, `OrderRecord` are not entities and are never flagged.
4. **Private and Internal Helper Methods:**
   - Private and protected helper methods (such as mapping functions `this::toDto` or `private Dto convert(Entity e)`) are not REST endpoints and are excluded from inspection.
5. **Security Principal Parameters:**
   - Method parameters annotated with `@AuthenticationPrincipal` or `@CurrentSecurityContext` are injected by Spring Security and are never treated as client-exposed input parameters.

---

### False Negative (FN) Handling & Defense-in-Depth
1. **Two-Pass Cross-File Resolution:**
   - Solves the AST file-ordering problem: if `OrderController.java` is parsed before `Order.java`, Pass 2 matches `Order` once all entity annotations have been indexed across the project.
2. **Deep Generic Unwrapping:**
   - Recursively unwraps complex generic responses: `ResponseEntity<List<UserEntity>>` correctly flags `UserEntity`.

---

## Code Examples

### Non-Compliant (Violating Anti-Pattern)

```java
// VIOLATION: Exposing JPA Entity directly in endpoint signature
@RestController
@RequestMapping("/api/users")
public class UserController {

    @PostMapping
    public ResponseEntity<User> createUser(@RequestBody User user) { // Mass-assignment vulnerability!
        return ResponseEntity.ok(userRepository.save(user));         // Lazy loading crash risk!
    }
}
```

### Compliant (Refactored Best Practice)

```java
// COMPLIANT: Using dedicated DTOs or Java 21 Records
@RestController
@RequestMapping("/api/users")
public class UserController {

    @PostMapping
    public ResponseEntity<UserResponse> createUser(@Valid @RequestBody CreateUserRequest request) {
        UserResponse response = userService.createUser(request);
        return ResponseEntity.ok(response);
    }
}
```

### Legitimate Edge Cases Handled Without Alerting

```java
// COMPLIANT: Spring MVC controller accepting standard framework Model
@Controller
public class WebPageController {
    @GetMapping("/dashboard")
    public String dashboard(Model model) { // Model is a framework UI object: permitted
        model.addAttribute("title", "Dashboard");
        return "dashboard";
    }
}
```

---

## Configuration & Tuning

In `archguard.yml`, you can customize rule severity, exclude specific classes, or adjust thresholds:

```yaml
rules:
  AG012:
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
@SuppressArchitecture("AG012")
public void myMethod() {
    // ...
}

// Or using standard Java @SuppressWarnings
@SuppressWarnings("archguard:AG012")
public void myMethod() {
    // ...
}
```
