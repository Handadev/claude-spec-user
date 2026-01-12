<!--
========================================
SYNC IMPACT REPORT
========================================
Version Change: (new) → 1.0.0
Ratification: 2026-01-12 (initial adoption)

Added Principles:
- I. Code Quality: Purity & Clarity
- II. Testing Standards: Design by Testing (TDD)
- III. Performance Requirements: Efficiency & Scalability
- IV. Single Source of Truth: Spec-First
- V. Security & Privacy by Design: Secure Default
- VI. Developer Experience (DX): Automation & Flow
- VII. Task & Progress Management: Visualization

Added Sections:
- Core Principles (7 principles)
- Governance

Removed Sections: N/A (initial creation)

Templates Requiring Updates:
- .specify/templates/plan-template.md ✅ updated (Constitution Check section aligned)
- .specify/templates/spec-template.md ✅ compatible (no changes needed)
- .specify/templates/tasks-template.md ✅ compatible (TDD workflow already present)

Follow-up TODOs: None
========================================
-->

# SpecUser Constitution

> **"The Spec is the Implementation, and the Test is the Design."**
>
> This constitution serves as an absolute promise to build a robust and scalable
> backend system for user management.

## Core Principles

### I. Code Quality: Purity & Clarity

> *"Code must be a clear narrative of the business logic."*

- **Ubiquitous Language**: Class names, method names, and variable names MUST
  perfectly match the terms defined in the API Specifications and Domain Model
  definitions. The use of synonyms is prohibited.

- **Spec-Based Type Strictness**: Conversions between DTOs and Entities MUST be
  explicit. Fields not defined in the API Spec (OpenAPI, etc.) MUST NOT be
  propagated into the internal logic.

- **Declarative Code**: Prefer Early Returns or Polymorphism over complex
  conditional statements (nested if-else) to ensure the code flow mirrors the
  logical flow of the specification.

### II. Testing Standards: Design by Testing (TDD)

> *"Write no production code without a failing test."*

- **Red-Green-Refactor Cycle (NON-NEGOTIABLE)**: Strictly adhere to the cycle:
  1. Write a failing test (Red) that verifies the spec
  2. Implement the minimum code to pass the test (Green)
  3. Improve the code (Refactor)

- **Spec = Test Case**: Test method names and verification logic MUST map 1:1
  with the "Functional Requirements" in the PRD or "Response Scenarios" in the
  API Spec.

- **Isolation & Contract**:
  - Unit Tests: Strictly mock external dependencies (DB, Network) to verify the
    purity of domain logic
  - Contract Tests: Use Spring Cloud Contract or Pact to automatically verify
    compliance with the API Spec for consumers

### III. Performance Requirements: Efficiency & Scalability

> *"Performance degradation is a functional failure."*

- **Query Efficiency Optimization**: MUST detect N+1 problems and verify index
  usage during the development phase (e.g., via Hibernate Query validation).

- **SLA Compliance**: MUST set target latency (e.g., P99 < 200ms) for each API
  in the spec. CI pipelines MUST block deployment if Load Tests fail to meet
  these targets.

- **Async & Caching**: Heavy tasks requiring no immediate response MUST be
  handled asynchronously (Event-Driven). Frequently read data MUST be cached
  according to a specified TTL policy.

### IV. Single Source of Truth: Spec-First

> *"The Spec is the Master; the Code is the Servant."*

- **Schema-First Development**: API development begins ONLY after the schema
  (YAML/Proto) is frozen. "Code-First" approaches are prohibited.

- **Spec-Implementation Synchronization**: The CI pipeline MUST break the build
  if discrepancies are detected between the actual implementation and the
  documented spec.

- **Change Control**: Changes to the Specification (PR) MUST precede or occur
  simultaneously with changes to the Code (PR).

### V. Security & Privacy by Design: Secure Default

> *"Assume every input is a potential attack."*

- **Strict Input Validation**: Regardless of frontend validation, the backend
  MUST strictly re-validate all requests based on constraints defined in the
  Spec (length, format, type).

- **Minimal Data Exposure**: Returning Entities directly is prohibited. Data
  MUST be converted to DTOs containing only the fields defined in the Spec to
  prevent sensitive data leakage.

- **Consistency in Auth/Authz**: All endpoints MUST be protected at the security
  filter level according to the Role/Permission policies defined in the Spec.

### VI. Developer Experience (DX): Automation & Flow

> *"Automate the repetitive; focus on the creative."*

- **Code Generation**: Boilerplate code (DTOs, Controller Interfaces, Client
  Code) MUST be auto-generated from the Spec (e.g., OpenAPI) to allow focus on
  business logic.

- **Reproducible Local Environment**: The local environment MUST perfectly mimic
  production (DB, Cache, Message Queue) using tools like docker-compose, runnable
  with a single command.

- **Living Documentation**: Configure the build so that test execution results
  automatically update API documentation snippets (e.g., Spring REST Docs).

### VII. Task & Progress Management: Visualization

> *"Work that is not recorded is work that is not done."*

- **Daily Task File Management**: At the start of work, check for the existence
  of a `task_{YYYY-MM-DD}.md` file in the root directory. If absent, create a
  new one. If present, append new tasks to the bottom of the existing list.

- **Taxonomy (Markdown Structure)**:
  - Header 2 (`##`) - Large Category: Core Business Feature (Epic/Feature)
  - Header 3 (`###`) - Medium Category: Unit Function (API Endpoint, Scheduler)
  - List Item (`-`) - Small Category: Specific Action Steps
  - MUST explicitly state the [Library/Pattern/Convention] to be used

- **Status Protocol**: Use Markdown checkboxes and Status Tags:
  - To Do: `- [ ] **(To Do)** Task Description`
  - In Progress: `- [ ] **(In Progress)** Task Description`
  - Done: `- [x] **(Done)** Task Description`

- **Completion & Commit Protocol**:
  1. Upon finishing a task, write a clear commit message and execute `git commit`
  2. Append `(Commit: {hash})` to the end of the task line in the .md file
  3. Change the checkbox to `[x]` to mark the task as fully complete

## Governance

- **Constitution Supremacy**: This constitution supersedes all other development
  practices. Any conflict MUST be resolved in favor of the constitution.

- **Amendment Procedure**:
  1. Propose changes via PR with clear rationale
  2. Amendments require documentation update, approval, and migration plan
  3. Version bump according to semantic versioning (MAJOR.MINOR.PATCH)

- **Compliance Review**: All PRs and code reviews MUST verify compliance with
  these principles. Complexity beyond these principles MUST be justified and
  documented.

- **Versioning Policy**:
  - MAJOR: Backward incompatible governance/principle removals or redefinitions
  - MINOR: New principle/section added or materially expanded guidance
  - PATCH: Clarifications, wording, typo fixes, non-semantic refinements

**Version**: 1.0.0 | **Ratified**: 2026-01-12 | **Last Amended**: 2026-01-12
