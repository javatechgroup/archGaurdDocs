# AG032: Broken Cryptographic Algorithm

| Attribute | Details |
| :--- | :--- |
| **Rule ID** | `AG032` |
| **Name** | Broken Cryptographic Algorithm |
| **Default Severity** | `ERROR` |
| **Category** | Security, Resilience & Safety |
| **Standard Mapping** | OWASP Top 10 (A02:2021 — Cryptographic Failures), CWE-327 |
| **Analysis Engine** | AST Source Analysis (JavaParser) |
| **Core Classes** | `AG032BrokenCryptoRule, SecurityVulnerabilityInspector` |

---

## Executive Summary & Architectural Rationale

Using obsolete or broken cryptographic hashing and encryption algorithms severely compromises data security:
1. **Collision Attacks:** MD5 and SHA-1 have known collision vulnerabilities, allowing attackers to generate different inputs that produce identical hash outputs (enabling forged digital signatures or certificates).
2. **Key Recovery Attacks:** Legacy symmetric ciphers like DES and 3DES use 56-bit or 112-bit effective keys that can be brute-forced in hours with modern hardware.
3. **Pattern Leakage in ECB Mode:** Electronic Codebook (ECB) encrypts identical plaintext blocks into identical ciphertext blocks, preserving data patterns and leaking confidential data structures.

---

## How It Is Implemented

AST inspection in `SecurityVulnerabilityInspector`:
1. **Broken Hash Algorithms:**
   - Detects `MessageDigest.getInstance(...)` with `MD5`, `MD2`, `SHA-1`, `SHA1`, or `SHA`.
2. **Weak Symmetric Ciphers:**
   - Detects `Cipher.getInstance(...)` with `DES`, `DESede`, `TripleDES`, `RC2`, `RC4`, `Blowfish`, or `ARCFOUR`.
3. **Insecure Electronic Codebook (ECB) Mode:**
   - Flags cipher transformations using ECB mode (e.g. `AES/ECB/PKCS5Padding` or any transformation containing `/ECB/`).

---

### False Positive (FP) Handling & Mitigations
1. **Modern Authenticated Algorithms:**
   - Modern algorithms such as `SHA-256`, `SHA-512`, `SHA3-256`, and `AES/GCM/NoPadding` are compliant and never flagged.
2. **Receiver Scope Verification:**
   - Only static factory calls on `MessageDigest` or `Cipher` are evaluated. Custom utility or factory classes with methods named `getInstance("MD5")` are never falsely flagged.
3. **Literal Parameter Matching:**
   - Mentions of "MD5" in log statements, comments, or HTTP header names (`Content-MD5`) are ignored; only explicit cryptographic factory invocations are evaluated.

---

## Code Examples

### Non-Compliant (Violating Anti-Pattern)

```java
// VIOLATION: Using collision-prone MD5 and DES in ECB mode
public class LegacyCryptoService {

    public byte[] hashData(byte[] input) throws Exception {
        MessageDigest md = MessageDigest.getInstance("MD5"); // VIOLATION: Obsolete hash!
        return md.digest(input);
    }

    public byte[] encrypt(byte[] key, byte[] data) throws Exception {
        Cipher cipher = Cipher.getInstance("AES/ECB/PKCS5Padding"); // VIOLATION: ECB mode leaks patterns!
        return cipher.doFinal(data);
    }
}
```

### Compliant (Refactored Best Practice)

```java
// COMPLIANT: Use SHA-256 and AES/GCM authenticated encryption
public class ModernCryptoService {

    public byte[] hashData(byte[] input) throws Exception {
        MessageDigest md = MessageDigest.getInstance("SHA-256");
        return md.digest(input);
    }

    public byte[] encrypt(SecretKey key, GCMParameterSpec spec, byte[] data) throws Exception {
        Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
        cipher.init(Cipher.ENCRYPT_MODE, key, spec);
        return cipher.doFinal(data);
    }
}
```
