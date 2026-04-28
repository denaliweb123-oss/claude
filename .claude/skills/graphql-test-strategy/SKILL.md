---
name: graphql-test-strategy
description: Analyzes a GraphQL API codebase and generates a production-grade TEST_STRATEGY.md covering testing levels, performance, security, automation, and tooling. Use when starting a new GraphQL project, preparing for a QA audit, or formalizing testing practices for engineering leadership review.
---

# GraphQL Test Strategy Generator

## Instructions

Act as a Senior QA Architect. Create a comprehensive GraphQL Test Strategy Document for a production-grade supergraph, written at a level of technical depth ready for engineering leadership review.

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

Write the document to the project root as `TEST_STRATEGY.md`. Use the following structure — include only sections relevant to the actual codebase; omit sections that don't apply:

```markdown
# Test Strategy Document
## GraphQL API — Production-Grade Test Strategy

**Role:** Senior QA Architect
**Scope:** [describe the API and its public URL if known]
**Context:** [deployment target, schema builder, filter engine, resource constraints]
**Audience:** Engineering Leadership

---

## 0. Strategic Intent & Discovery

### 0.1 Discovery Questions
Write each as an explicit question. Follow each with: *Answer: [what was found and how it shaped testing].*
- Data integrity: how the source of truth is validated through schema transformations
- Resource constraints: what limits apply (memory, CPU, rate limits)
- Consumer safety: how breaking changes are prevented for public consumers

### 0.2 Priority Scenarios (Risk-Based)
List the 3–5 highest-risk scenarios with a concrete failure mode, severity rating (Critical / High / Medium), and blast radius.

### 0.3 Out of Scope & Trade-offs
Each exclusion must name the item, the reason for deferral (time / cost / feasibility), and the proxy signal that will substitute.

### 0.4 AI Methodology
Document the model used, the exact prompt, what the AI produced, and specific changes made during human review.

---

## 1. Schema Governance & Type Safety

Cover:
- Code-first vs SDL-first approach in this project
- Automated breaking change detection: tool used (graphql-inspector, Rover Subgraph Checks), CI integration, and exit criteria
- Schema snapshot testing for response structure regression
- Client code generation if applicable
- Schema linting rules (graphql-eslint)

---

## 2. Testing Levels

### 2.1 Unit Tests — Resolver Logic
- Extract complex logic from resolvers into standalone functions; test those directly with Jest or Bun test
- Cover: field splitting, type coercion, null guards, validation helpers
- Tooling: [name the test runner used in this project]

### 2.2 Integration Tests — Schema-Driven
- Execute full GraphQL operations against the in-memory server (yoga.fetch, Apollo test utils, etc.)
- Key scenarios: happy paths, nullability edge cases, filter combinations, error paths
- Snapshot testing: capture JSON response shapes and fail CI on structural drift

### 2.3 Contract Testing
- Schema diffing strategy (graphql-inspector diff against main/production)
- Exit criteria: zero breaking changes without explicit approval and version bump

### 2.4 End-to-End Tests — Critical Flows
- Define the 3–5 highest-value user flows that must work end-to-end
- Tooling: Playwright or Cypress against a deployed staging environment
- Trigger: pre-release gate only (not on every PR)

---

## 3. Security

Cover only what is relevant to the actual schema and deployment:
- **Introspection**: disabled in production (`NODE_ENV=production`), enabled for authenticated developers in non-production
- **Query depth limiting**: hard limit value and plugin used (e.g., `useDepthLimit({ maxDepth: N })`)
- **Query complexity analysis**: cost budget, plugin used (e.g., `graphql-query-complexity`), and what triggers a block
- **Rate limiting**: requests/minute threshold and enforcement layer (CDN, middleware, or plugin)
- **ReDoS prevention**: validation applied to user-supplied regex inputs before execution
- **Field-level authorization**: scope-based plugin and which fields are gated (if auth is present)

---

## 4. Performance

- **N+1 detection**: how N+1 patterns are identified in tests (query counts, tracing assertions)
- **DataLoader strategy**: which resolvers use DataLoaders, batching window, and cache scope
- **Load testing**: tool (k6, Artillery), target throughput, and latency thresholds for critical queries
- **Cache verification**: how cache-hit rates are validated (headers, tracing, or observability tooling)
- **Observability**: tracing and monitoring tooling (Apollo GraphOS, Stellate, or equivalent)

---

## 5. Automation & CI/CD

### 5.1 Exit Criteria for Release
List concrete, measurable gates with current status:
| Gate | Target | Status |

### 5.2 Workflow Steps
Ordered list of CI steps grounded in the actual pipeline config. Recommended order:
1. Lint & type check
2. Schema diff (breaking change detection)
3. Unit & integration tests
4. Schema snapshot validation
5. Security checks (depth, complexity, ReDoS fuzz)
6. E2E tests (staging, pre-release only)

---

## 6. Tooling Recommendations

List the recommended tools per concern, distinguishing between currently in use and recommended additions:
| Concern | Tool | Status |
|---|---|---|
| Functional testing | Jest / Bun test | [In use / Recommended] |
| Schema validation | graphql-inspector or Rover | [In use / Recommended] |
| Monitoring & tracing | Apollo GraphOS | [In use / Recommended] |
| Load testing | k6 or Artillery | [In use / Recommended] |
| E2E | Playwright | [In use / Recommended] |
| Query complexity | graphql-query-complexity | [In use / Recommended] |

---

## 7. Implementation Checklist

### Already Implemented
Use `[x]` only for items verified against the source (cite file:line).

### Recommended (Not Yet Implemented)
Use `[ ]` for all items not confirmed in source. Do not guess.
```

### Step 4: Review Before Saving

Before writing the file:
- Confirm every `[x]` checklist item by citing the specific file and line number that implements it. If you cannot verify it, mark it `[ ]`.
- Remove any section that has no real content for this project — do not pad with placeholder text.
- Ensure all resource constraints (memory limits, CPU budgets, rate limits) are specific numbers sourced from config files, not generic placeholders.
- Security and performance sections must reflect only what is found or concretely recommended for this codebase — not generic industry patterns.
- Tone must be technical and precise, suitable for an engineering leadership review.

Write the final document to `TEST_STRATEGY.md` in the project root.
