# AG030: Missing HTTP Client Timeout Configuration

| Attribute | Details |
| :--- | :--- |
| **Rule ID** | `AG030` |
| **Name** | Missing HTTP Client Timeout Configuration |
| **Default Severity** | `ERROR` |
| **Category** | Cloud Resilience & Microservice Reliability |
| **Analysis Engine** | AST Source Analysis (JavaParser) |
| **Core Classes** | `AG030HttpClientTimeoutRule, HttpClientTimeoutInspector` |

---

## Executive Summary & Architectural Rationale

In Spring Boot and standard Java, default HTTP clients use infinite ($0$ ms) timeouts:
1. **Infinite Hang & Worker Thread Starvation:** When downstream microservices, external payment gateways, or third-party APIs experience network stalls or outages, calling an unconfigured HTTP client blocks the caller thread indefinitely.
2. **Cascading Microservice Outages:** As incoming HTTP requests pile up, all Tomcat or Jetty worker threads block waiting on stalled HTTP clients. Within seconds, the calling service exhausts all worker threads, stops responding to health checks (`/actuator/health`), and collapses, cascading failures across the entire cluster.
3. **Silent Time Bombs:** Code works fine in development where latency is 0ms, but causes total outages in cloud environments during minor network hiccups.

---

## How It Is Implemented

AST inspection in `HttpClientTimeoutInspector`:
1. **Java 11+ `HttpClient`:**
   - Detects `HttpClient.newHttpClient()`, which creates an immutable client lacking connection timeouts.
2. **Spring 6+ `RestClient`:**
   - Detects `RestClient.create()` with zero arguments, which falls back to unconfigured `SimpleClientHttpRequestFactory`.
   - Inspects `RestClient.builder().build()` to verify whether `.requestFactory()` or customizers configuring timeouts are present.
3. **`RestTemplate`:**
   - Detects direct `new RestTemplate()` instantiation in Spring components.
   - For `@Bean` methods creating `RestTemplate`: checks if `setRequestFactory()` is invoked on the instance before returning.
   - For `RestTemplateBuilder.build()`: checks if `setConnectTimeout()`, `setReadTimeout()`, `requestFactory()`, or `apply()` are configured.

---

### False Positive (FP) Handling & Mitigations
1. **Configured `RestTemplateBuilder`:**
   - Calls to `builder.setConnectTimeout(...).setReadTimeout(...).build()` are recognized as properly configured and are not flagged.
2. **Configured Request Factory:**
   - In `@Bean` methods, if `restTemplate.setRequestFactory(...)` or `HttpComponentsClientHttpRequestFactory` is configured, AG030 recognises the factory configuration and exempts the bean.
3. **`RestClient` with Custom Request Factory:**
   - `RestClient.builder().requestFactory(...)` is compliant.
4. **Spring 6+ HTTP Service Proxy Factories:**
   - Calls to `HttpServiceProxyFactory.builderFor(...)` or `HttpServiceProxyFactory.builder(...)` using `RestClientAdapter` build declarative HTTP interface proxy factories rather than unconfigured HTTP clients, and are not falsely flagged.
5. **Non-HTTP or Third-Party Builders:**
   - Generic builders (e.g. `TemplateBuilder`, `JsonMapper.builder()`, `OkHttpClient.Builder`, or custom service builders taking a `RestClient` parameter) are differentiated from `RestClient.Builder` / `RestTemplateBuilder` to eliminate false positives.

---

### False Negative (FN) Handling & Defense-in-Depth
1. **Direct `new RestTemplate()` in Beans:**
   - Catches developers directly writing `RestTemplate rt = new RestTemplate();` inside `@Service` or `@Component` classes without timeout configuration.
2. **Factory Method Return Type Matching:**
   - Inspects builder methods even when builder variable names vary (`builder`, `restTemplateBuilder`, `customBuilder`).

---

## Code Examples

### Non-Compliant (Violating Anti-Pattern)

```java
// VIOLATION: HTTP clients created with default infinite timeouts
@Configuration
public class ClientConfig {

    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate(); // INFINITE DEFAULT TIMEOUT! Thread starvation risk.
    }

    @Bean
    public RestClient restClient() {
        return RestClient.create(); // Infinite default timeout!
    }

    public void callRemote() {
        HttpClient client = HttpClient.newHttpClient(); // Immutable client without connect timeout!
    }
}
```

### Compliant (Refactored Best Practice)

```java
// COMPLIANT: Explicit connection and read timeouts configured
@Configuration
public class ClientConfig {

    @Bean
    public RestTemplate restTemplate(RestTemplateBuilder builder) {
        return builder
                .setConnectTimeout(Duration.ofSeconds(3))
                .setReadTimeout(Duration.ofSeconds(5))
                .build();
    }

    @Bean
    public RestClient restClient() {
        SimpleClientHttpRequestFactory factory = new SimpleClientHttpRequestFactory();
        factory.setConnectTimeout(Duration.ofSeconds(3));
        factory.setReadTimeout(Duration.ofSeconds(5));

        return RestClient.builder()
                .requestFactory(factory)
                .build();
    }

    public HttpClient httpClient() {
        return HttpClient.newBuilder()
                .connectTimeout(Duration.ofSeconds(3))
                .build();
    }
}
```

### Legitimate Edge Cases Handled Without Alerting

```java
// COMPLIANT: RestTemplate with setRequestFactory in @Bean method
@Bean
public RestTemplate customRestTemplate() {
    SimpleClientHttpRequestFactory factory = new SimpleClientHttpRequestFactory();
    factory.setConnectTimeout(3000);
    factory.setReadTimeout(5000);
    RestTemplate rt = new RestTemplate();
    rt.setRequestFactory(factory); // Explicit factory configured: compliant
    return rt;
}
```

---

## Configuration & Tuning

In `archguard.yml`, you can customize rule severity, exclude specific classes, or adjust thresholds:

```yaml
rules:
  AG030:
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
@SuppressArchitecture("AG030")
public void myMethod() {
    // ...
}

// Or using standard Java @SuppressWarnings
@SuppressWarnings("archguard:AG030")
public void myMethod() {
    // ...
}
```
