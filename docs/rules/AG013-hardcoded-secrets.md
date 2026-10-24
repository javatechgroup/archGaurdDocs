# AG013: Hardcoded Credentials & Secrets

| Attribute | Details |
| :--- | :--- |
| **Rule ID** | `AG013` |
| **Name** | Hardcoded Credentials & Secrets |
| **Default Severity** | `ERROR` |
| **Category** | Application Security |
| **Analysis Engine** | AST Source Analysis (Pattern & Entropy Scanning) |
| **Core Classes** | `AG013HardcodedSecretRule, SecretsScannerInspector` |

---

## Executive Summary & Architectural Rationale

Hardcoding credentials in source code is a critical security vulnerability:
1. **Credential Theft:** Committing secrets to Git repositories exposes them to unauthorized users, CI/CD logs, and compromised developer machines.
2. **Inability to Rotate:** Hardcoded credentials cannot be rotated without rebuilding, re-testing, and redeploying production binaries.
3. **Compliance Violations:** Fails SOC2, PCI-DSS, ISO 27001, and HIPAA compliance audits.

---

## How It Is Implemented

AST analysis inspects variable declarations and string literals in `SecretsScannerInspector`:
1. **Regex Pattern Matching on String Literals:**
   - AWS Access Key IDs: `\b(AKIA[0-9A-Z]{16})\b`
   - Private Keys: `-----BEGIN (?:[A-Z ]+)?PRIVATE KEY-----`
   - GitHub Personal Access Tokens: `\b(gh[pousr]_[0-9a-zA-Z]{36})\b`
   - Slack OAuth Tokens: `\b(xox[baprs]-[0-9a-zA-Z]{10,48})\b`
2. **Sensitive Variable Name Heuristics:**
   - Scans variables matching: `password`, `passwd`, `secret`, `apikey`, `accesstoken`, `jwtsecret`, `privatekey`, `clientsecret`.
   - Verifies variable initializer is a string literal and satisfies `isLikelyRealSecret()`.
3. **Report Masking:** All detected secrets are masked (`AKIA...****`) in CLI, SARIF, and HTML outputs to avoid leaking credentials into reports.

---

### False Positive (FP) Handling & Mitigations
1. **Non-Secret Variable Suffixes:**
   - Variables ending in: `header`, `url`, `uri`, `path`, `endpoint`, `param`, `parameter`, `name`, `keyname`, `field`, `prop`, `property`, `regex`, `pattern`, `prompt`, `label`, `message`, `title`, `description`, `placeholder` are explicitly excluded (e.g. `String apiKeyHeader = "X-API-KEY";` is NOT flagged).
2. **Spring Property Placeholders:**
   - Values formatted like `"${app.security.jwt-secret}"` are configuration injection keys, not hardcoded secrets.
3. **URL Paths and HTTP Headers:**
   - Strings starting with `/`, `http://`, `https://`, `X-`, or matching `Authorization` are ignored.
4. **Test & Dummy Fallback Values:**
   - Standard test strings: `password`, `admin`, `root`, `test`, `demo`, `dummy`, `sample`, `localhost`, `127.0.0.1`, `changeit`, `todo` are ignored.

---

### False Negative (FN) Handling & Defense-in-Depth
1. **Standalone Literal Pattern Matching:**
   - High-entropy tokens (AWS, GitHub, Slack) are detected even if assigned to variables with innocent names (e.g. `String token = "AKIAIOSFODNN7EXAMPLE";`).
2. **Case-Insensitive Variable Names:**
   - Matches snake_case (`api_key`), camelCase (`apiKey`), and kebab-case naming variations.

---

## Code Examples

### Non-Compliant (Violating Anti-Pattern)

```java
// VIOLATION: Hardcoded AWS key and database password
public class PaymentClient {

    private String awsKey = "AKIA1234567890ABCDEF"; // AWS credential detected!

    private String dbPassword = "SuperSecretPassword123!"; // Sensitive variable with secret!
}
```

### Compliant (Refactored Best Practice)

```java
// COMPLIANT: Inject credentials from environment or Vault
@Component
public class PaymentClient {

    @Value("${aws.access.key}")
    private String awsKey;

    @Value("${db.password}")
    private String dbPassword;
}
```

### Legitimate Edge Cases Handled Without Alerting

```java
// COMPLIANT: Header name constant with 'key' in variable name
public class HeaderConstants {
    public static final String API_KEY_HEADER = "X-API-KEY"; // Exempted non-secret suffix
}
```

---

## Configuration & Tuning

In `archguard.yml`, you can customize rule severity, exclude specific classes, or adjust thresholds:

```yaml
rules:
  AG013:
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
@SuppressArchitecture("AG013")
public void myMethod() {
    // ...
}

// Or using standard Java @SuppressWarnings
@SuppressWarnings("archguard:AG013")
public void myMethod() {
    // ...
}
```
