# AG033: Insecure CORS Configuration

| Attribute | Details |
| :--- | :--- |
| **Rule ID** | `AG033` |
| **Name** | Insecure CORS Configuration |
| **Default Severity** | `ERROR` |
| **Category** | Security, Resilience & Safety |
| **Standard Mapping** | OWASP Top 10 (A05:2021 — Security Misconfiguration), CWE-942 |
| **Analysis Engine** | AST Source Analysis (JavaParser) |
| **Core Classes** | `AG033InsecureCorsRule, SecurityVulnerabilityInspector` |

---

## Executive Summary & Architectural Rationale

Cross-Origin Resource Sharing (CORS) defines how web browsers allow scripts running on one origin to access resources from a different origin:
1. **Arbitrary Origin Authentication:** Enabling `allowCredentials(true)` while setting the allowed origin to wildcard `*` enables any malicious website visited by an authenticated user to perform requests with the user's session cookies and read sensitive responses.
2. **Browser Rejection & Runtime Errors:** W3C CORS specifications explicitly disallow `Access-Control-Allow-Origin: *` when credentials are supported. Browsers will reject responses from such endpoints, creating subtle frontend integration bugs.
3. **Session Hijacking & Data Exfiltration:** Malicious cross-origin scripts can steal user profile data, financial accounts, or internal API responses.

---

## How It Is Implemented

AST inspection in `SecurityVulnerabilityInspector`:
1. **`@CrossOrigin` Annotations:**
   - Detects `@CrossOrigin` on classes and controller methods where `origins` (or `value`) includes `"*"` (or is defaulted) AND `allowCredentials = "true"`.
2. **Programmatic `CorsRegistry` Configuration:**
   - Detects call chains on `CorsRegistry` / `CorsRegistration` chaining `.allowedOrigins("*")` with `.allowCredentials(true)`.

---

### False Positive (FP) Handling & Mitigations
1. **Public Read-Only Endpoints:**
   - `@CrossOrigin(origins = "*")` without `allowCredentials` is compliant and **never flagged** (standard practice for public APIs).
2. **Specific Trusted Origins & Class Inheritance:**
   - `@CrossOrigin(origins = "https://app.example.com", allowCredentials = "true")` specifies a strict domain whitelist and is compliant.
   - When a class sets explicit trusted origins via `@CrossOrigin(origins = "https://...")`, method-level `@CrossOrigin(allowCredentials = "true")` inherits the class-level trusted origins and is **never falsely flagged** as a wildcard origin.

---

## Code Examples

### Non-Compliant (Violating Anti-Pattern)

```java
// VIOLATION: Wildcard origins combined with credential forwarding
@RestController
@CrossOrigin(origins = "*", allowCredentials = "true") // VIOLATION!
public class AccountController {

    @GetMapping("/api/account")
    public AccountDetails getDetails() {
        return accountService.getCurrentUserAccount();
    }
}
```

### Compliant (Refactored Best Practice)

```java
// COMPLIANT: Restrict allowed origins to specific trusted domains
@RestController
@CrossOrigin(origins = "https://app.mycompany.com", allowCredentials = "true")
public class AccountController {

    @GetMapping("/api/account")
    public AccountDetails getDetails() {
        return accountService.getCurrentUserAccount();
    }
}
```
