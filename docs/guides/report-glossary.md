# 📊 ArchGuard Report Indicators, Badges & Glossary

> **A comprehensive visual reference guide explaining every severity level, status tag, impact metric, and badge displayed in ArchGuard reports.**

---

## 📑 Table of Contents
1. [Severity Levels (ERROR vs WARNING vs INFO)](#1-severity-levels)
2. [Origin & Baseline Statuses (NEW vs BASELINED vs RESOLVED)](#2-origin--baseline-statuses)
3. [Blast Radius & Architectural Coupling](#3-blast-radius--coupling)
4. [Pull Request (PR) Risk Rating & Verdicts](#4-pr-risk-rating--verdicts)
5. [Architectural Health Score (A to F)](#5-architectural-health-score)
6. [Component Layer Badges](#6-component-layer-badges)

---

## 1. Severity Levels

ArchGuard classifies every architectural rule finding into one of three severities:

| Severity | Color / Badge | What It Means | Build Impact (Default) | Action Required |
| :--- | :---: | :--- | :---: | :--- |
| **`ERROR`** | 🔴 Red | **Critical Architectural or Safety Defect.**<br>Bypasses architectural boundaries (e.g. Controller $\to$ Repository), breaks Spring proxy contracts (self-invocation), leaks database connections (remote I/O in `@Transactional`), causes concurrency starvation, or introduces security vulnerabilities (SQL injection, broken crypto, insecure CORS). | **FAILS THE BUILD** ❌<br>(if newly introduced) | **Must be fixed** before code can be merged into the main branch. |
| **`WARNING`** | 🟡 Yellow | **Code Quality & Maintainability Smell.**<br>Anti-patterns that do not immediately crash the application, but accumulate technical debt, impede testability, or increase maintenance costs (e.g. Field `@Autowired`, `System.out.println`, complex methods, unpaginated collection returns, or dependency bloat). | **DOES NOT FAIL BUILD** ⚠️<br>(reported as informational) | Recommended to refactor during normal development. (Can be promoted to fail build via `-Darchguard.failOnSeverity=WARNING`). |
| **`INFO`** | 🔵 Blue | **Best Practice / Optimization Opportunity.**<br>Minor architectural notices, naming convention observations, or performance suggestions. | **DOES NOT FAIL BUILD** ℹ️ | Clean up when convenient; zero operational risk. |

---

## 2. Origin & Baseline Statuses

When analyzing code, ArchGuard compares findings against the version-controlled `.architecture-guard-baseline.json` snapshot to separate newly introduced debt from pre-existing legacy issues:

| Status Tag | Visual Badge | Meaning | Build Failure? |
| :--- | :---: | :--- | :---: |
| **`NEW`** | <span style="background:#dc2626; color:#fff; padding:2px 6px; border-radius:4px; font-weight:bold;">🔴 NEW</span> | **Introduced in Current Changes.**<br>This violation was newly added in the current commit or pull request. | **YES**<br>(if severity $\ge$ `failOnSeverity`) |
| **`BASELINED`** | <span style="background:#475569; color:#fff; padding:2px 6px; border-radius:4px; font-weight:bold;">🛡️ BASELINED</span> | **Pre-existing Legacy Technical Debt.**<br>This violation existed before ArchGuard was adopted and was frozen in the baseline snapshot. | **NO**<br>(exempted from failing the build) |
| **`RESOLVED`** | <span style="background:#16a34a; color:#fff; padding:2px 6px; border-radius:4px; font-weight:bold;">✅ RESOLVED</span> | **Fixed Technical Debt.**<br>A legacy violation recorded in the baseline file has been successfully fixed or refactored away. In ratchet mode (`--ratchet`), it is automatically pruned from the baseline. | **NO**<br>(celebrated as quality improvement) |
| **`SUPPRESSED IN CODE`** | <span style="background:#0284c7; color:#fff; padding:2px 6px; border-radius:4px; font-weight:bold;">🛡️ SUPPRESSED IN CODE</span> | **Audited Exception.**<br>A developer added an explicit `@SuppressWarnings("archguard:...")` annotation on the class or method with a justified business reason. | **NO**<br>(auditable in the dedicated dashboard tab) |

---

## 3. Blast Radius & Coupling

The **Blast Radius** badge indicates **how many other classes directly depend on this component** (also known as **Fan-In / Afferent Coupling $C_a$**).

> [!NOTE]
> **Blast Radius is NOT an error or violation!** Having many callers is completely normal and expected for core entities (`Booking`, `User`, `Order`) and central services (`EmailService`). It is an **informational risk indicator** to help developers prioritize testing and review.

| Callers (Dependents) | Tier | Badge | Architectural Meaning |
| :---: | :---: | :---: | :--- |
| **0 – 1** | **Low** | 🟢 **1 Caller (Low)** | **Isolated Component:** Leaf node (e.g. a specific controller or private helper). Changes or bugs only affect this single isolated spot. |
| **2 – 4** | **Moderate** | 🟡 **3 Callers (Moderate)** | **Shared Component:** Reused across a small group of related features. Changes carry moderate regression risk. |
| **$\ge$ 5** | **High** | 🔴 **💥 N Callers (High Blast Radius)** | **Central Core Component:** A foundational building block of the application (e.g. `Booking` with 14 callers). Any bug, schema modification, or performance bottleneck will ripple across all 14 callers simultaneously. |

### Why This Matters in Practice:
- **Triage Priority:** If you have 5 violations in a report, fix violations on **High Blast Radius** classes first, because fixing them protects the largest portion of your application.
- **Review Diligence:** When reviewing a Pull Request that modifies a High Blast Radius class, ensure integration tests cover all dependent features before approving.

---

## 4. PR Risk Rating & Verdicts

ArchGuard automatically analyzes the **delta** (new code in the current PR) and computes an overall Risk Rating:

| Risk Rating | Visual Pill | Automated PR Verdict | Criteria |
| :--- | :---: | :---: | :--- |
| **High Risk** | 🔴 **HIGH RISK** | `CHANGES REQUESTED ❌` | PR introduces 1 or more new `ERROR` violations, security hazards (OWASP), or touches high blast radius classes with unmitigated violations. |
| **Moderate Risk** | 🟡 **MODERATE RISK** | `APPROVED WITH COMMENTS ⚠️` | PR introduces `WARNING` severity smells (e.g. missing validation, field injection) with zero critical errors. |
| **Low Risk** | 🟢 **LOW RISK** | `APPROVED ✅` | PR satisfies all architectural rules, introduces zero new violations, or only resolves existing technical debt. |

---

## 5. Architectural Health Score

The **Health Score** provides an executive-level percentage rating (0% to 100%) and letter grade reflecting overall codebase compliance:

$$\text{Health Score} = \max\left(0, 100 - (\text{Errors} \times 5 + \text{Warnings} \times 1)\right)$$

| Grade | Score Range | Meaning | Recommendation |
| :---: | :---: | :--- | :--- |
| **A** | **90% – 100%** | **Excellent Architectural Hygiene.** Minimal or zero technical debt. | Maintain current quality gates. |
| **B** | **80% – 89%** | **Good Architecture.** A few minor warnings or isolated code smells. | Address warnings during sprint maintenance. |
| **C** | **70% – 79%** | **Moderate Architectural Decay.** Noticeable accumulation of layer skips or smells. | Schedule a refactoring spike. |
| **D** | **60% – 69%** | **Substantial Architectural Erosion.** Multiple critical errors and anti-patterns. | Freeze debt with baseline immediately; enforce quality gate on all new PRs. |
| **F** | **< 60%** | **Severe Architectural Hazard.** High risk of production outages and regression cascades. | Architectural intervention required. |

---

## 6. Component Layer Badges

ArchGuard automatically classifies every compiled class into its respective architectural layer using Spring stereotypes, JPA annotations, and package naming patterns:

| Badge | Typical Annotations / Conventions | Allowed Dependencies (Outbound) |
| :--- | :--- | :--- |
| **`CONTROLLER`** | `@RestController`, `@Controller` | May call `SERVICE`. Must **never** call `REPOSITORY` or directly return `ENTITY`. |
| **`SERVICE`** | `@Service` | May call other `SERVICE` beans, `REPOSITORY`, and external clients. Must **never** call `CONTROLLER`. |
| **`REPOSITORY`** | `@Repository`, Spring Data interfaces | Data access layer. Must **never** call `SERVICE` or `CONTROLLER`. |
| **`ENTITY`** | `@Entity`, `@Table` | JPA persistence models. Should be mapped to DTOs/Records before crossing the API boundary. |
| **`CONFIG`** | `@Configuration` | Spring configuration infrastructure beans. |
| **`DOMAIN`** | Pure Java domain models, Records | Business logic domain objects; isolated from infrastructure frameworks. |
| **`SECURITY`** | `SecurityFilterChain`, `AuthenticationProvider` | Authentication and authorization boundary. |
| **`CLIENT`** | `RestTemplate`, `RestClient`, Feign, WebClient | External outbound HTTP/RPC integrations. |
| **`UTIL`** | Stateless utility helper classes | General-purpose shared functions. |
