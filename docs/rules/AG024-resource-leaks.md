# AG024: AutoCloseable Resource Leaks

| Attribute | Details |
| :--- | :--- |
| **Rule ID** | `AG024` |
| **Name** | AutoCloseable Resource Leaks |
| **Default Severity** | `ERROR` |
| **Category** | Resource Management |
| **Analysis Engine** | AST Source Analysis (JavaParser) |
| **Core Classes** | `AG024ResourceLeakRule, ResourceAndSafetyInspector, AstUtils` |

---

## Executive Summary & Architectural Rationale

Failure to release system resources is an insidious production stability killer:
1. **File Descriptor Exhaustion:** Operating systems limit open file descriptors per process (`ulimit -n`). Unclosed `FileInputStream` or `Socket` instances leak descriptors until the OS refuses all new file opens and HTTP connections, requiring a process restart.
2. **HikariCP Connection Pool Starvation:** JDBC `Connection`, `Statement`, or `ResultSet` instances left open without closing will leak database connections until the entire connection pool blocks permanently.
3. **Direct Memory Leaks:** Native I/O buffers remain allocated in direct memory outside the JVM heap.

---

## How It Is Implemented

AST analysis inspects variable declarations in method bodies:
1. Indexes all resources declared in `TryStmt.getResources()` (try-with-resources header).
2. Inspects declared variable types against `AUTOCLOSEABLE_RESOURCE_TYPES`:
   - `FileInputStream`, `FileOutputStream`, `FileReader`, `FileWriter`
   - `BufferedReader`, `BufferedWriter`, `InputStream`, `OutputStream`
   - `Connection`, `Statement`, `PreparedStatement`, `CallableStatement`, `ResultSet`
   - `Socket`, `ServerSocket`, `ZipInputStream`, `GZIPInputStream`
3. Checks if the resource is an in-memory stream via `AstUtils.isInMemoryResource()`:
   - Excludes `ByteArrayInputStream`, `ByteArrayOutputStream`, `StringReader`, `StringWriter`.
4. If an AutoCloseable OS resource is instantiated as a local variable outside a try-with-resources block, records `ResourceLeakFinding`.

---

### False Positive (FP) Handling & Mitigations
1. **Standard Try-With-Resources:**
   - Any resource declared in `try (InputStream is = ...)` is automatically closed by the Java compiler at the end of the block and is completely compliant.
2. **Spring Managed Resources:**
   - Resources managed by Spring (e.g. `JdbcTemplate`, `ResourceLoader`) handle lifecycle automatically and do not declare raw JDBC connection variables.
3. **In-Memory Heap Streams and Buffers:**
   - In-memory buffers backed solely by JVM heap arrays (`ByteArrayInputStream`, `ByteArrayOutputStream`, `StringReader`, `StringWriter`, `CharArrayReader`, `CharArrayWriter`) do not allocate or hold operating system file descriptors, sockets, or native memory. According to official JDK documentation, their `close()` method is a no-op. Instantiating in-memory streams outside try-with-resources is completely safe and is exempted.

---

### False Negative (FN) Handling & Defense-in-Depth
1. **Traditional Try-Finally Manual Closes:**
   - Traditional `try { ... } finally { is.close(); }` blocks are prone to suppressed exceptions and errors if `close()` throws. AG024 strictly enforces modern try-with-resources.
2. **Multiple Resources in Single Method:**
   - Tracks every AutoCloseable variable independently.

---

## Code Examples

### Non-Compliant (Violating Anti-Pattern)

```java
// VIOLATION: InputStream opened outside try-with-resources
public String readFile(String path) throws IOException {
    FileInputStream fis = new FileInputStream(path); // LEAK: File descriptor never closed!
    byte[] data = fis.readAllBytes();
    return new String(data);
}
```

### Compliant (Refactored Best Practice)

```java
// COMPLIANT: Modern Java try-with-resources
public String readFile(String path) throws IOException {
    try (FileInputStream fis = new FileInputStream(path)) { // Guaranteed close
        return new String(fis.readAllBytes());
    }
}

// ALSO COMPLIANT: Using Java NIO Files
public String readFileNio(Path path) throws IOException {
    return Files.readString(path); // Internal resource safety
}
```

### Legitimate Edge Cases Handled Without Alerting

```java
// COMPLIANT: Multiple resources in try-with-resources
try (InputStream is = Files.newInputStream(path);
     BufferedReader reader = new BufferedReader(new InputStreamReader(is))) {
    return reader.readLine();
}

// COMPLIANT: In-memory stream (ByteArrayInputStream) does not leak OS file descriptors
byte[] decodedBytes = Base64.getDecoder().decode(base64Payload);
InputStream inputStream = new ByteArrayInputStream(decodedBytes); // Permitted: heap-allocated in-memory buffer
```

---

## Configuration & Tuning

In `archguard.yml`, you can customize rule severity, exclude specific classes, or adjust thresholds:

```yaml
rules:
  AG024:
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
@SuppressArchitecture("AG024")
public void myMethod() {
    // ...
}

// Or using standard Java @SuppressWarnings
@SuppressWarnings("archguard:AG024")
public void myMethod() {
    // ...
}
```
