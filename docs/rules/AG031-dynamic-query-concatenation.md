# AG031: Dynamic Query Concatenation Injection Risk

| Attribute | Details |
| :--- | :--- |
| **Rule ID** | `AG031` |
| **Name** | Dynamic Query Concatenation Injection Risk |
| **Default Severity** | `ERROR` |
| **Category** | Security, Resilience & Safety |
| **Standard Mapping** | OWASP Top 10 (A03:2021 — Injection), CWE-89 |
| **Analysis Engine** | AST Source Analysis (JavaParser) |
| **Core Classes** | `AG031DynamicQueryConcatenationRule, SecurityVulnerabilityInspector` |

---

## Executive Summary & Architectural Rationale

Concatenating dynamic runtime variables directly into SQL or JPQL statements is a primary cause of injection vulnerabilities:
1. **SQL / JPQL Injection:** Direct string concatenation allows untrusted user inputs to alter SQL query structure, resulting in unauthorized data leakage, tampering, or administrative privilege escalation.
2. **Bypassing Query Plan Caching:** Dynamic queries force the database to recompile query execution plans on every execution, causing CPU churn and performance degradation.
3. **Loss of Static Type Checking:** Dynamic string queries circumvent Spring Data and JPA compile-time query syntax validation.

---

## How It Is Implemented

AST inspection in `SecurityVulnerabilityInspector`:
1. **Spring Data `@Query` Annotations:**
   - Inspects `@Query` annotations on repository methods.
   - Detects dynamic variable concatenation using binary `+` operators.
   - Compile-time constant strings formatted across multiple lines are recognized via constant folding and exempted.
2. **Persistence Execution APIs:**
   - Detects dynamic string concatenation in `EntityManager.createQuery(...)`, `EntityManager.createNativeQuery(...)`.
   - Detects dynamic string concatenation in Spring `JdbcTemplate` methods (`query`, `queryForObject`, `queryForList`, `update`, `execute`).
   - Detects dynamic string concatenation in JDBC `Connection.prepareStatement(...)`.
3. **SQL Keyword Guardrails:**
   - Validates that the query string contains SQL/JPQL statement keywords (`SELECT`, `INSERT`, `UPDATE`, `DELETE`, `FROM`, `WHERE`, `MERGE`) to prevent false positives on general methods named `update` or `execute`.

---

### False Positive (FP) Handling & Mitigations
1. **Multiline Constant String Concatenation & Text Blocks:**
   - Queries formatted across multiple lines using literal strings or modern Java 15+ text blocks (`"""..."""`) (e.g. `"""SELECT u FROM User u """ + "WHERE u.active = true"`) are recognized as compile-time constants and are **never flagged**.
2. **Static Constant Compositions:**
   - Standard static constants defined in `ALL_CAPS` (e.g. `BASE_QUERY + WHERE_ACTIVE`) are recognized as constant compositions and exempted.
3. **Non-SQL Domain Methods:**
   - Domain or service methods named `execute` or `update` operating on non-persistence beans (e.g. `cacheService.update(...)`, `auditLogger.execute(...)`) without actual SQL targets are ignored.

---

## Code Examples

### Non-Compliant (Violating Anti-Pattern)

```java
// VIOLATION: Concatenating user input into JPQL / SQL queries
@Repository
public interface UserRepository extends JpaRepository<User, Long> {

    @Query("SELECT u FROM User u WHERE u.username = '" + input + "'") // VIOLATION!
    User findByUsername(String input);
}

@Service
public class OrderService {
    @Autowired
    private EntityManager em;

    public List<Order> searchOrders(String customerId) {
        // VIOLATION: SQL injection risk via string concatenation
        return em.createQuery("SELECT o FROM Order o WHERE o.customerId = '" + customerId + "'")
                 .getResultList();
    }
}
```

### Compliant (Refactored Best Practice)

```java
// COMPLIANT: Use named parameters or positional binding
@Repository
public interface UserRepository extends JpaRepository<User, Long> {

    @Query("SELECT u FROM User u WHERE u.username = :username")
    User findByUsername(@Param("username") String username);
}

@Service
public class OrderService {
    @Autowired
    private EntityManager em;

    public List<Order> searchOrders(String customerId) {
        return em.createQuery("SELECT o FROM Order o WHERE o.customerId = :customerId", Order.class)
                 .setParameter("customerId", customerId)
                 .getResultList();
    }
}
```

### Legitimate Edge Cases Handled Without Alerting

```java
// COMPLIANT: Multiline formatted constant SQL query is not flagged
@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {

    @Query("SELECT p FROM Product p " +
           "WHERE p.inStock = true " +
           "ORDER BY p.name ASC")
    List<Product> findAvailableProducts();
}
```
