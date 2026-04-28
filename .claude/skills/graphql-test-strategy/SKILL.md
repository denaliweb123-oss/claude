---
name: graphql-test-strategy
description: Analyzes a GraphQL API codebase and generates a comprehensive TEST_STRATEGY.md document. Use when starting a new GraphQL project, preparing for a QA audit, or formalizing testing practices.
---

# GraphQL Test Strategy Generator

## Instructions

### Step 1: Explore the Codebase

Before writing anything, gather the following context by reading relevant files:

- **Schema**: GraphQL schema file(s) or the code-first schema builder (e.g., Pothos, Nexus, type-graphql). Identify all types, queries, mutations, subscriptions, and field-level nullability.
- **Existing tests**: Any test files — note what is already covered and what tooling is used (Jest, Vitest, Bun test, etc.).
- **Resolvers**: How resolvers are implemented, whether DataLoaders are used, and where N+1 patterns may exist.
- **Data sources**: Databases, external APIs, in-memory packages. Note what serves as the "source of truth."
- **Infrastructure/deployment**: Deployment target (Cloudflare Workers, Lambda, Node.js server, etc.) and any resource constraints (memory limits, cold starts, rate limits).
- **Security config**: Auth plugins, depth limiting, query complexity, introspection settings, rate limiting.
- **CI/CD**: Existing GitHub Actions or pipeline config files to understand what already runs in CI.
- **README / CLAUDE.md**: For any domain-specific context or constraints already documented.

### Step 2: Identify Risk Areas

Based on the codebase exploration, identify:

1. **Complexity hotspots**: Deeply nested relationships that could exhaust memory or trigger N+1.
2. **Filter/input attack surface**: Any user-supplied regex, nested input objects, or dynamic query building.
3. **Public contract surface**: Fields/types that are consumed by external clients and must not break.
4. **Nullability gaps**: Fields that should be nullable but are not (or vice versa) — these cause runtime errors.
5. **Missing coverage**: Resolvers or edge cases with no existing tests.

### Step 3: Generate TEST_STRATEGY.md

Write the document to the project root as `TEST_STRATEGY.md`. Structure it as follows — include only sections relevant to the actual codebase; omit sections that don't apply:

```markdown
# Test Strategy Document
## GraphQL API Testing & AI-Augmented Quality Engineering

**Role:** [infer from context or use "Principal Software Engineer"]
**Scope:** [describe the API and its public URL if known]
**Context:** [deployment target, schema builder, filter engine, resource constraints]

---

## 0. Strategic Intent & Discovery

### 0.1 Discovery Questions
- Data integrity: how the source of truth is validated through schema transformations
- Resource constraints: what limits apply (memory, CPU, rate limits)
- Consumer safety: how breaking changes are prevented for public consumers

### 0.2 Priority Scenarios (Risk-Based)
List the 2–4 highest-risk scenarios discovered in Step 2, with brief justification.

### 0.3 Out of Scope & Trade-offs
What is explicitly deferred and why (e.g., load testing, full i18n coverage).

---

## 1. Schema Governance & Type Safety

Cover:
- Code-first vs SDL-first approach in this project
- How breaking changes are detected (graphql-inspector, schema snapshots, etc.)
- Client code generation if applicable
- Schema linting rules (graphql-eslint)

---

## 2. The Testing Pyramid

### 2.1 Resolver Unit Tests
- What logic is isolated and tested independently
- Which utilities/helpers have unit tests
- Tooling (Bun test, Jest, Vitest)

### 2.2 Integration Tests
- How full operations are tested (yoga.fetch, Apollo test utils, etc.)
- Key scenarios: happy paths, nullability edge cases, filter combinations
- Whether a real or mock data source is used

### 2.2.1 Snapshot Testing (if applicable)
- Structural regression detection for complex query responses

### 2.3 Contract Testing
- Schema diffing strategy (graphql-inspector diff against main/production)
- Exit criteria: zero breaking changes without explicit approval

### 2.4 End-to-End Tests (if applicable)
- Client-facing scenarios, tooling (Playwright, Cypress)

---

## 2.5 AI-Augmented Testing (if applicable)
- Query fuzzing, adversarial filter inputs, synthetic data generation

---

## 3. Security & DoS Prevention

Cover only what is relevant to the actual schema and deployment:
- Query depth limiting (hard limit value, plugin used)
- Query cost analysis / complexity budgets
- ReDoS prevention for regex inputs
- Introspection control per environment
- Field-level authorization (if auth is present)

---

## 4. Performance & Efficiency

- N+1 prevention strategy (DataLoader, batching)
- How N+1 is detected in tests
- Tracing/observability tooling if present

---

## 5. CI/CD Pipeline Integration

### 5.1 Exit Criteria for Release
List concrete, measurable gates (e.g., zero breaking changes, coverage threshold, fuzz pass rate).

### 5.2 Workflow Steps
Ordered list of CI steps, grounded in the actual pipeline config found in the repo.

---

## 6. Implementation Checklist

Use `[x]` for items already implemented (confirmed by reading the code) and `[ ]` for items recommended but not yet present. Be precise — only mark `[x]` if you verified it in the source.
```

### Step 4: Review Before Saving

Before writing the file:
- Confirm every `[x]` checklist item by pointing to the specific file/line that implements it.
- Remove any section that has no real content for this project (don't pad with placeholder text).
- Ensure resource constraints (memory limits, rate limits) are specific numbers, not generic placeholders.
- Make the security section reflect actual plugins/middleware found in the codebase, not aspirational ones.

Write the final document to `TEST_STRATEGY.md` in the project root.
