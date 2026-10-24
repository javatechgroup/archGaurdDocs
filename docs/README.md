# Architecture Guard Documentation Hub

Welcome to the comprehensive documentation suite for **Java Application Quality Guard (ArchGuard)**.

---

## Documentation Directory

### 1. Guides (`docs/guides/`)
- **[User Manual](guides/user-manual.md)**: End-to-end user manual covering installation, Maven configuration, multi-thread sizing (`<threads>`), rule severity tuning, baseline debt ratcheting, CLI switches, and HTML dashboard navigation.
- **[Configuration Reference](guides/configuration.md)**: Full reference for `archguard.yml`, thread pool tuning (`analysis.threads`), package classification patterns, custom package boundaries, and rule threshold customizations.
- **[Release Guide](guides/release-guide.md)**: Release checklist, Maven Central deployment guide, GPG artifact signing, and versioning instructions.

### 2. Technical Architecture & Internals (`docs/technical/`)
- **[System Architecture Guide](technical/architecture.md)**: In-depth technical architecture, C4 container & component topology, centralized multithreaded concurrency engine (`ArchGuardTaskExecutor`), dual-engine internals (ArchUnit ASM bytecode engine + JavaParser AST engine), and Tarjan's SCC cycle detection algorithms.
- **[Developer Extensibility Guide](technical/developer-guide.md)**: Guide for core contributors and architects on how to author custom static analysis rules, create custom AST `SourceInspector`s, and register custom violations.
- **[Feature Matrix & Comparison](technical/features.md)**: Detailed feature breakdown and comparative analysis against ArchUnit, SonarQube, and Checkstyle.

### 3. Specifications & Requirements (`docs/specifications/`)
- **[Software Requirements Specification (SRS)](specifications/requirements.md)**: Complete functional (FR) and non-functional (NFR) engineering specifications.

### 4. Governance & Architectural Rules (`docs/rules/`)
- **[Complete Rules Catalog & Deep-Dive Guides](rules/README.md)**: Dedicated technical specifications for every single rule (`AG001` - `AG033`), detailing AST/bytecode implementation, false positive (FP) heuristics, false negative (FN) defenses, and refactoring patterns.
  - Layer Enforcement (`AG001` - `AG004`)
  - Cycle & Boundary Detection (`AG005` - `AG006`)
  - Clean Code & Injection (`AG007`, `AG017`, `AG020`)
  - Error Handling & Reliability (`AG008`, `AG023`, `AG024`)
  - Concurrency & Thread Safety (`AG010`, `AG022`, `AG027`, `AG028`, `AG029`)
  - Database & Performance (`AG009`, `AG014`, `AG019`, `AG025`, `AG026`)
  - Security & API Governance (`AG011`, `AG012`, `AG013`, `AG015`, `AG016`, `AG030`, `AG031`, `AG032`, `AG033`)
- **[Rules Summary Reference](rules/rules-summary.md)**: Quick rule-by-rule reference table with descriptions and default severities.

### 5. Articles & Publications (`docs/articles/`)
- **[Technical LinkedIn Article](articles/LINKEDIN_ARTICLE.md)**
- **[PDF Article: Guarding Enterprise Spring Boot Architecture](articles/Guarding-Spring-Boot-Architecture-Article.pdf)**
- **[Java Application Quality Guard Document](articles/Java-Application-Quality-Guard.docx)**
