# Architecture Guard Rules Reference Catalog

This directory contains detailed technical documentation for all **33 built-in rules** (AG001 – AG033) supported by **Architecture Guard**.
Each rule document provides the complete implementation details, false positive (FP) heuristics, false negative (FN) defenses, and code examples.

---

## Complete Rules Index

| ID | Rule Name | Category | Severity | Engine | Documentation |
| :--- | :--- | :--- | :---: | :--- | :--- |
| `AG001` | **Controller to Repository Direct Dependency** | Layer Boundary Enforcement | `ERROR` | Bytecode Analysis | [AG001-controller-to-repository.md](./AG001-controller-to-repository.md) |
| `AG002` | **Controller to Controller Direct Dependency** | Layer Boundary Enforcement | `ERROR` | Bytecode Analysis | [AG002-controller-to-controller.md](./AG002-controller-to-controller.md) |
| `AG003` | **Service to Controller Inverted Dependency** | Inverted Layer Hierarchy | `ERROR` | Bytecode Analysis | [AG003-service-to-controller.md](./AG003-service-to-controller.md) |
| `AG004` | **Repository to Service Inverted Dependency** | Inverted Layer Hierarchy | `ERROR` | Bytecode Analysis | [AG004-repository-to-service.md](./AG004-repository-to-service.md) |
| `AG005` | **Class-Level Cyclic Dependency** | Coupling & Maintainability | `ERROR` | Graph Theory | [AG005-cyclic-dependency.md](./AG005-cyclic-dependency.md) |
| `AG006` | **Package Boundary Violation** | Custom Architectural Boundaries | `WARNING` | Bytecode Analysis | [AG006-package-boundary.md](./AG006-package-boundary.md) |
| `AG007` | **Prohibited Field Injection** | Clean Code & Testability | `WARNING` | AST Source Analysis | [AG007-field-injection.md](./AG007-field-injection.md) |
| `AG008` | **Swallowed Exceptions** | Error Handling & Reliability | `WARNING` | AST Source Analysis | [AG008-swallowed-exceptions.md](./AG008-swallowed-exceptions.md) |
| `AG009` | **Missing @Transactional on Multi-Repository Writes** | Data Integrity & Transactions | `WARNING` | AST Source Analysis | [AG009-missing-transactional.md](./AG009-missing-transactional.md) |
| `AG010` | **Unsafe Mutable Static State** | Concurrency & Thread Safety | `ERROR` | AST Source Analysis | [AG010-unsafe-static-state.md](./AG010-unsafe-static-state.md) |
| `AG011` | **Direct Console Logging** | Production Observability | `WARNING` | AST Source Analysis | [AG011-direct-console-logging.md](./AG011-direct-console-logging.md) |
| `AG012` | **Direct Entity Exposure in REST Controllers** | Security & API Governance | `ERROR` | AST Source Analysis | [AG012-direct-entity-exposure.md](./AG012-direct-entity-exposure.md) |
| `AG013` | **Hardcoded Credentials & Secrets** | Application Security | `ERROR` | AST Source Analysis | [AG013-hardcoded-secrets.md](./AG013-hardcoded-secrets.md) |
| `AG014` | **JPA Eager Fetch Detection** | Database Performance | `WARNING` | AST Source Analysis | [AG014-jpa-eager-fetch.md](./AG014-jpa-eager-fetch.md) |
| `AG015` | **Missing @Valid on REST @RequestBody** | API Resilience | `WARNING` | AST Source Analysis | [AG015-missing-validation.md](./AG015-missing-validation.md) |
| `AG016` | **Unpaginated REST Collections** | Memory Safety & API Scaling | `WARNING` | AST Source Analysis | [AG016-unpaginated-collections.md](./AG016-unpaginated-collections.md) |
| `AG017` | **Injected Dependency Bloat / God Class** | Maintainability & Modularity | `WARNING` | AST Source Analysis | [AG017-dependency-bloat.md](./AG017-dependency-bloat.md) |
| `AG018` | **High Cyclomatic Complexity** | Maintainability & Code Quality | `WARNING` | AST Source Analysis | [AG018-cyclomatic-complexity.md](./AG018-cyclomatic-complexity.md) |
| `AG019` | **Algorithmic Performance Risk in Loops** | Runtime Performance | `WARNING` | AST Source Analysis | [AG019-algorithmic-performance-loops.md](./AG019-algorithmic-performance-loops.md) |
| `AG020` | **Code Readability & Nesting** | Clean Code | `WARNING` | AST Source Analysis | [AG020-code-readability-nesting.md](./AG020-code-readability-nesting.md) |
| `AG021` | **Spring Proxy Self-Invocation** | Spring Runtime Traps | `ERROR` | AST Source Analysis | [AG021-proxy-self-invocation.md](./AG021-proxy-self-invocation.md) |
| `AG022` | **Singleton Mutable Instance State** | Concurrency & Thread Safety | `ERROR` | AST Source Analysis | [AG022-singleton-mutable-state.md](./AG022-singleton-mutable-state.md) |
| `AG023` | **NullPointer Traps & Defensive Coding** | Defensive Programming | `WARNING` | AST Source Analysis | [AG023-null-safety.md](./AG023-null-safety.md) |
| `AG024` | **AutoCloseable Resource Leaks** | Resource Management | `ERROR` | AST Source Analysis | [AG024-resource-leaks.md](./AG024-resource-leaks.md) |
| `AG025` | **Loop String Concatenation** | Performance & Memory | `WARNING` | AST Source Analysis | [AG025-loop-string-concatenation.md](./AG025-loop-string-concatenation.md) |
| `AG026` | **Remote I/O in @Transactional Boundary** | Database Resilience | `ERROR` | AST Source Analysis | [AG026-remote-io-in-transaction.md](./AG026-remote-io-in-transaction.md) |
| `AG027` | **Java 21 Virtual Thread Pinning** | Modern Java Concurrency | `WARNING` | AST Source Analysis + Target Java Version Gate | [AG027-virtual-thread-pinning.md](./AG027-virtual-thread-pinning.md) |
| `AG028` | **ThreadLocal Memory Leak** | Memory Safety & Concurrency | `ERROR` | AST Source Analysis | [AG028-thread-local-leak.md](./AG028-thread-local-leak.md) |
| `AG029` | **Unbounded @Async Thread Pool** | Threading & Resource Safety | `WARNING` | AST Source Analysis | [AG029-unbounded-async.md](./AG029-unbounded-async.md) |
| `AG030` | **Missing HTTP Client Timeout Configuration** | Cloud Resilience & Microservice Reliability | `ERROR` | AST Source Analysis | [AG030-http-client-timeout.md](./AG030-http-client-timeout.md) |
| `AG031` | **Dynamic Query Concatenation Injection Risk** | Security, Resilience & Safety | `ERROR` | AST Source Analysis | [AG031-dynamic-query-concatenation.md](./AG031-dynamic-query-concatenation.md) |
| `AG032` | **Broken Cryptographic Algorithm** | Security, Resilience & Safety | `ERROR` | AST Source Analysis | [AG032-broken-crypto.md](./AG032-broken-crypto.md) |
| `AG033` | **Insecure CORS Configuration** | Security, Resilience & Safety | `ERROR` | AST Source Analysis | [AG033-insecure-cors.md](./AG033-insecure-cors.md) |

---

## Dual-Engine Architecture Overview

Architecture Guard combines **two complementary static analysis engines** orchestrated concurrently to achieve high accuracy with zero false positives:

1. **Bytecode Analysis Engine (ArchUnit + ASM + Tarjan's SCC Graph):**
   - **Rules Governed:** `AG001` – `AG006`.
   - **Why Bytecode?** Operates on compiled `.class` files, providing exact type resolution across classpath dependencies without wildcard ambiguity. Extracts the complete directed dependency graph (inheritance, interfaces, fields, constructor parameters, method arguments, return types, and invocations) to detect architectural layer inversions and cyclic dependencies in $O(V+E)$ time.

2. **AST Source Code Analysis Engine (JavaParser + Modular SourceInspectors):**
   - **Rules Governed:** `AG007` – `AG033`.
   - **Why AST?** Inspects Java source files directly before bytecode stripping occurs. Recovers annotations, parameter names, catch clauses, raw literals, loop structures, and local try-with-resources scopes to catch code smells, thread safety hazards, resource leaks, and modern Java 21 Project Loom traps.

3. **Centralized Concurrent Execution (`ArchGuardTaskExecutor`):**
   - **High-Throughput Parallel Orchestration:** Both engines execute **concurrently** via an isolated `ForkJoinPool` with named daemon threads (`archguard-worker-N`), cutting wall-clock execution time by 40–60%.
   - **Thread-Safe AST Parallelism:** Source files are distributed across worker threads using `ThreadLocal<JavaParser>` instances and lock-free thread-safe accumulators, processing thousands of files in parallel while preserving strict deterministic finding order.
   - **Configurable Thread Sizing:** Tunable via `<threads>N</threads>` in POM, `analysis.threads` in `archguard.yml`, or `-Darchguard.threads=N` on the CLI (setting `threads: 1` activates deterministic single-threaded caller execution).

---

## False Positive (FP) & False Negative (FN) Mitigation Philosophy

| Mitigation Strategy | Technical Implementation | Rules Benefiting |
| :--- | :--- | :--- |
| **Two-Pass Symbol Resolution** | Pass 1 gathers project-wide declarations (entities, executor beans); Pass 2 resolves forward references. | `AG012`, `AG029` |
| **3-Tier Layer Classification** | Explicit stereotypes > Configured package regex > Suffix heuristics. | `AG001` – `AG004` |
| **Tarjan's SCC Canonical Rotation** | Normalizes cycles to lexicographically lowest starting class, reporting 1 canonical cycle. | `AG005` |
| **Modern Immutability Recognition** | Unpacks `List.of()`, `Collections.unmodifiable*()`, `Stream.toList()`, Guava builders. | `AG010`, `AG022` |
| **Language & Target Version Gating** | Automatically deactivates carrier pinning warnings on target Java 24+ (JEP 491). | `AG027` |
| **Type Disambiguation in Loops** | Disambiguates $O(1)$ `Set.contains()` and `String.contains()` from $O(n)$ `List.contains()`. | `AG019`, `AG025` |
| **Lifecycle Context Awareness** | Distinguishes paired interceptor methods (`preHandle` & `afterCompletion`). | `AG028` |
| **Framework Type Whitelisting** | Whitelists MVC `Model`, `BindingResult`, streaming byte arrays, and HTTP entity wrappers. | `AG012`, `AG015` |
| **Selective Read-Only Filtering** | Excludes queries (`find*`, `get*`, `count*`) from transaction write counts. | `AG009` |
| **Standard Intentional Convention** | Respects catch parameters named `ignored` or `expected`. | `AG008` |
| **Arity-Aware Signature Matching** | Distinguishes overloaded methods by parameter counts (`name#arity`). | `AG021` |
