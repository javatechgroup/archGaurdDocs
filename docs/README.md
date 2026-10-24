# Architecture Guard Documentation Hub

Welcome to the comprehensive documentation suite for **Java Application Quality Guard (ArchGuard)**.

---

## Documentation Directory

### 1. Guides (`docs/guides/`)
- **[User Manual](guides/user-manual.md)**: End-to-end user manual covering installation, build tool configuration (Maven & Gradle), multi-thread sizing (`<threads>`), rule severity tuning, baseline debt ratcheting, CLI switches, and HTML dashboard navigation.
- **[Configuration Reference](guides/configuration.md)**: Full reference for `archguard.yml`, thread pool tuning (`analysis.threads`), package classification patterns, custom package boundaries, and rule threshold customizations.

### 2. Features & Capabilities (`docs/technical/`)
- **[Feature Matrix & Comparison](technical/features.md)**: Detailed feature breakdown, PR code review bot, and comparative benchmark analysis against ArchUnit and SonarQube.

### 3. Governance & Architectural Rules (`docs/rules/`)
- **[Complete Rules Catalog & Deep-Dive Guides](rules/README.md)**: Dedicated technical specifications for every single rule (`AG001` - `AG033`), detailing detection heuristics, code examples, and refactoring patterns:
  - Layer Enforcement (`AG001` - `AG004`)
  - Cycle & Boundary Detection (`AG005` - `AG006`)
  - Clean Code & Injection (`AG007`, `AG017`, `AG020`)
  - Error Handling & Reliability (`AG008`, `AG023`, `AG024`)
  - Concurrency & Thread Safety (`AG010`, `AG022`, `AG027`, `AG028`, `AG029`)
  - Database & Performance (`AG009`, `AG014`, `AG019`, `AG025`, `AG026`)
  - Security & API Governance (`AG011`, `AG012`, `AG013`, `AG015`, `AG016`, `AG030`, `AG031`, `AG032`, `AG033`)
- **[Rules Summary Reference](rules/rules-summary.md)**: Quick rule-by-rule reference table with descriptions and default severities.

### 4. Articles & Publications (`docs/articles/`)
- **[Technical LinkedIn Article](articles/LINKEDIN_ARTICLE.md)**
- **[PDF Article: Guarding Enterprise Spring Boot Architecture](articles/Guarding-Spring-Boot-Architecture-Article.pdf)**

