---
name: graphql-strategy-audit
description: Audits a GraphQL Test Strategy document against five production-grade categories (schema integrity, security, performance, operational health, environment parity). Use when reviewing an existing TEST_STRATEGY.md for completeness, clarity, and risk coverage.
---

# GraphQL Strategy Auditor

## Instructions

Act as a Senior QA Architect and Security Auditor. Audit the strategy document against production-grade standards across five categories below.

### Step 1: Ask for the File

Ask the user:

> "What is the file name or path of the strategy document to audit? (default: `TEST_STRATEGY.md` in the project root)"

- If the user provides a path, use it as-is.
- If the user presses enter or says "default", use `TEST_STRATEGY.md` from the project root.

Read the file. If it does not exist at the given path, stop and tell the user — suggest generating one with the `graphql-test-strategy` skill if no file exists yet.

### Step 2: Ask About Custom Requirements

Before running the audit, ask the user:

> "Do you have any custom requirements to audit against, in addition to the five standard ones? (yes / no)"

- If the user says **yes**: ask them to share their custom requirements. Wait for their input. Append each custom requirement to the audit as additional numbered entries (6, 7, etc.), applying the same Met / Partially Met / Missed rating and one suggestion per entry.
- If the user says **no**: proceed directly to Step 3 with only the five standard categories.

### Step 3: Audit Against Production-Grade Standards

For each category, identify **Gaps**, **Risks**, and **Recommended Actions**. Then assign one of: **Met**, **Partially Met**, or **Missed**.

**Category 1 — Schema Integrity**
Does the strategy include automated breaking change detection (e.g., using Rover Subgraph Checks or `graphql-inspector`) and snapshot testing for response structures?

- Met: Both breaking change detection (with a named tool and CI integration) and snapshot testing are explicitly covered.
- Partially Met: One of the two is covered, or both are mentioned without specifics on tooling or CI integration.
- Missed: No schema diffing or snapshot strategy is documented.

**Category 2 — Security**
Does the strategy explicitly cover: disabling introspection in production, implementing query complexity/depth limits, and verifying field-level authorization?

- Met: All three are addressed with concrete configuration details or test scenarios.
- Partially Met: One or two are addressed; the third is absent or mentioned without specifics.
- Missed: Security concerns are generic or entirely absent.

**Category 3 — Performance**
Are there strategies for detecting N+1 problems (e.g., using DataLoader) and verifying cache-hit rates?

- Met: Both N+1 detection (with a named approach such as DataLoader or query tracing) and cache verification are explicitly documented.
- Partially Met: N+1 is addressed but cache strategy is absent, or vice versa.
- Missed: No performance testing strategy is documented.

**Category 4 — Operational Health**
Does the plan verify error-masking (hiding stack traces in production) and the existence of structured audit logs?

- Met: Both error-masking validation and structured audit log verification are explicitly covered.
- Partially Met: One is addressed; the other is missing.
- Missed: No operational health concerns are documented.

**Category 5 — Environment Parity**
Are integration tests validated against multiple environment variants (dev, staging, prod)?

- Met: The strategy explicitly names the environments tested and how parity is enforced.
- Partially Met: Multiple environments are mentioned but without a parity strategy or named test targets.
- Missed: Tests are only described against a single environment or no environment is specified.

**Custom Categories (if provided)**
For each custom category supplied by the user, identify Gaps, Risks, and Recommended Actions using the same three-tier rating (Met / Partially Met / Missed). Number them starting at 6.

### Step 4: Write the Audit Report

Output a structured report in this format:

```
## GraphQL Strategy Audit Report

| Category | Status | Gaps | Risks | Recommended Actions |
|---|---|---|---|---|
| 1. Schema Integrity | [Met / Partially Met / Missed] | [gaps] | [risks] | [actions] |
| 2. Security | [Met / Partially Met / Missed] | [gaps] | [risks] | [actions] |
| 3. Performance | [Met / Partially Met / Missed] | [gaps] | [risks] | [actions] |
| 4. Operational Health | [Met / Partially Met / Missed] | [gaps] | [risks] | [actions] |
| 5. Environment Parity | [Met / Partially Met / Missed] | [gaps] | [risks] | [actions] |
| 6. [Custom category name] | [Met / Partially Met / Missed] | [gaps] | [risks] | [actions] |

### Overall Assessment
[2–3 sentences on the document's strongest area and the single highest-priority gap to address.]
```

Omit row 6 (and beyond) if no custom categories were provided.

Be specific: reference actual section names, line content, or missing items from the document rather than giving generic feedback.
