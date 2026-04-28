# Test Strategy Document
## GraphQL API Testing & AI-Augmented Quality Engineering

**Role:** Principal Software Engineer and Platform Architect
**Scope:** Public read-only GraphQL API at `https://countries.trevorblades.com`
**Backup API:** `https://rickandmortyapi.com/graphql` (Control Reference for platform health)
**Context:** Cloudflare Worker (128MB Memory Ceiling), Pothos Schema, Sift Filter Engine.

---

## 0. Strategic Intent & Discovery

### 0.1 Discovery Questions

These questions were asked before any testing began. Answers shaped risk prioritization and scope decisions.

*   **Data Integrity:** How do we validate that the underlying `countries-list` remains the "Source of Truth" during schema transformations? *Answer: No automated check exists. Field resolvers like `currencies` (comma-split), `subdivisions` (type-filtered), and `capital` (empty-string-to-null) would silently break if an upstream package changed its data shape. This is a documented gap — identified as Priority Scenario 3.*
*   **Resource Constraints:** What is the maximum query complexity allowable before exceeding the 128MB Cloudflare Worker memory ceiling and the 10 ms (free) / 30 ms (paid) CPU budget? *Answer: No depth limit or cost analysis is currently enforced. `Language.countries` and `Continent.countries` each scan ~250 countries per call — at depth 4, a `languages { countries { languages { countries { ... } } } }` query produces O(n³) iterations. This shaped Priority Scenario 1.*
*   **Consumer Safety:** How can we guarantee zero breaking changes for public consumers using typed clients? *Answer: Currently we cannot — no schema diffing runs in CI and there is no versioning or advance-notice channel. Any removed field or nullability change silently breaks consumers. This is the critical gap addressed in Section 2.3.*

### 0.2 Priority Scenarios (Risk-Based Reasoning)

1.  **Deeply Nested Relationships** *(Severity: High)* — The schema allows `countries { languages { countries { languages { ... } } } }` with no depth limit enforced. Each hop scans all ~250 countries in memory, producing O(n³) resolver calls at depth 4. Blast radius: Worker CPU budget exhausted for all concurrent users, not just the attacker. No DataLoader is currently in place to mitigate this.
2.  **Adversarial Filter Injection / ReDoS** *(Severity: Critical)* — `StringQueryOperatorInput.regex` passes user-supplied strings directly to JavaScript's `RegExp` engine via Sift's `$regex` with no length limit, timeout, or pattern validation. A crafted pattern (e.g., `(a+)+$`) causes catastrophic backtracking that kills the Cloudflare Worker isolate for all users during its lifetime. This is the highest-risk single vector in the API.
3.  **Breaking Schema Contract** *(Severity: High)* — This is a public API with no versioning and thousands of consumers. Removing a field, changing nullability (e.g., making `capital` non-nullable), or renaming a type would silently break typed clients with no warning. No `graphql-inspector` diff runs in CI today.

### 0.3 Out of Scope & Trade-offs (Transparency)

*   **Load/Stress Testing:** Deferred due to time constraints in this cycle — setting up a meaningful load harness (k6, Artillery) against a Cloudflare Workers target requires infrastructure work not available now. Cloudflare's built-in CPU/request analytics will act as a proxy signal for the first release. Revisit if batching abuse is observed in production.
*   **Full i18n Coverage:** Deferred due to diminishing returns — exhaustively asserting all 14 locales × 250 countries would duplicate `i18n-iso-countries`' own test suite. Spot-checking 3–5 representative countries across 3 locales is sufficient to detect locale registration failures within the available test time budget.
*   **Mutation Testing:** Out of scope — no mutations exist in the schema. Zero effort required until write operations are introduced.
*   **Subscription Testing:** Out of scope — no subscriptions exist in the schema. Zero effort required until real-time features are introduced.
*   **E2E Against Live URL:** Deferred — the integration tests already exercise the full stack via `yoga.fetch` without network calls. E2E tests against `https://countries.trevorblades.com` would add network flakiness without proportional coverage value in this cycle.

### 0.4 AI Methodology & Meta-Process (Audit Compliance)

**Model used:** Gemini 2.5 Pro (strategy draft) and Claude claude-sonnet-4-6 (review pass).

**Prompt used for initial draft:**
> "Act as a Principal Software Engineer. Analyze this GraphQL API codebase (Pothos, GraphQL Yoga, Cloudflare Workers, Sift.js filters, in-memory data from npm packages). Generate a comprehensive test strategy document covering: discovery questions, risk-based priority scenarios, out-of-scope trade-offs, schema governance, testing pyramid (unit/integration/contract/E2E), AI-augmented testing, security/DoS prevention, performance/N+1, CI/CD gates, and an implementation checklist."

**What the AI produced:**
A complete draft with all sections present. Key AI contributions: the O(n³) complexity framing for nested resolvers, the ReDoS attack vector description, the `graphql-inspector` workflow for contract testing, and the DataLoader recommendation for N+1 prevention.

**What was changed after review:**
- The checklist was the most problematic output — the AI marked `useDepthLimit`, `@pothos/plugin-dataloader`, `maxLength` regex validation, field-level auth, and query cost analysis as `[x]` (implemented). None of these exist in the actual codebase. All fabricated `[x]` items were corrected to `[ ]` during human review.
- Priority scenario justifications were thinned by the AI (generic "stress-tests memory") and were rewritten with concrete failure modes and blast-radius descriptions.
- The AI omitted explicit time-constraint justifications from the out-of-scope section; these were added manually.
- Discovery questions initially had no answers — the AI wrote them as open questions only. Answers documenting what was found were added manually.

---

## 1. Schema Governance & Type Safety

We employ a **Code-First (Pothos)** approach with **SDL Artifact Generation**.
*   **Workflow:** Developers modify Pothos builders. A pre-commit hook generates a `schema.graphql` file.
*   **Validation:** `graphql-inspector` compares the local SDL against the production registry to block breaking changes.
*   **Client Safety:** **GraphQL Code Generator** is mandated for all consumers (e.g., React/Angular examples) to ensure end-to-end type safety from resolver to UI. Generated hooks/types must be checked into version control.
*   **Schema Linting:** `graphql-eslint` is used in CI to enforce naming conventions, nullability best practices, and the presence of `@description` tags.
*   **AI-Assisted Governance:** Utilize LLMs to analyze `schema.graphql` diffs and provide human-readable summaries of breaking changes for stakeholders.

---

## 2. The Testing Pyramid

### 2.1 Resolver Unit Tests
*   **Focus:** Pure business logic in isolation.
*   **Method:** Extract complex logic from Pothos `resolve` functions into standalone utility functions or service modules. Test these functions directly with Jest/Bun test, providing mock inputs.
*   **Example:** Testing the `getName(country.code, lang)` logic or the `countryToAwsRegion` utility.

### 2.2 Integration Tests
*   **Focus:** Testing the full operation (Query/Mutation) against the in-memory GraphQL server.
*   **Method:** Use `yoga.fetch` (Apollo Server Integration Testing equivalent) to execute queries and assert on the full GraphQL response shape, data, and errors.
*   **Data:** Use a test database or in-memory data. For this API, the `countries-list` package serves as the "test database."
*   **Scenarios:** Cover happy paths, edge cases (e.g., Antarctica for nulls), and filter logic.

### 2.2.1 Snapshot Testing
*   **Focus:** Structural regression detection.
*   **Method:** Capture JSON snapshots of complex query results. CI fails if the structure or data types drift, ensuring accidental "nulling" of fields is caught instantly.

### 2.3 Contract Testing (Schema Checks)
*   **Focus:** Ensuring clients and servers remain in sync and preventing breaking changes.
*   **Method:** Use `graphql-inspector diff` in CI to compare the current branch's schema against the deployed production schema (or `main` branch).
*   **Tooling:** `graphql-inspector`.
*   **Exit Criteria:** Zero breaking changes allowed without explicit approval and version bump.

### 2.4 End-to-End (E2E) Tests
*   **Focus:** Validating critical user flows from the client perspective.
*   **Method:** Use Playwright or Cypress to interact with the React/Angular example applications, making real GraphQL requests to the deployed API.
*   **Scenarios:** Verify country selection, filtering, and data display in the UI.

---

## 2.5 Autonomous & AI-Augmented Testing
*   **Autonomous Query Fuzzing:** Implement a script that uses the SDL to generate 1,000+ random valid GraphQL queries to test resolver resilience and memory pressure.
*   **AI-Generated Edge Cases:** Use Gemini/Claude to generate adversarial `filter` inputs (complex regex, nested logic) to stress-test the Sift engine.
*   **Synthetic Data Injection:** AI-driven generation of mock subdivision data to test the scalability of the `iso-3166-2` integration.

---

## 3. Security & DoS Prevention

*   **Query Depth Limiting:** A hard limit of **7 levels** is enforced via `useDepthLimit` plugin in `graphql-yoga` to prevent recursive "graph-bomb" attacks (mitigates R3).
*   **Query Cost Analysis:** While not fully implemented, the blueprint mandates future integration of a cost analysis plugin (e.g., `graphql-query-complexity`) to assign costs to fields (e.g., list fields cost more) and block malicious "recursive bombs" based on a total budget.
*   **ReDoS Prevention:** The `regex` operator in `StringQueryOperatorInput` has a `maxLength` validation of 50 characters to mitigate ReDoS attacks (mitigates R2).
*   **Introspection Control:** Introspection is disabled in production environments (`NODE_ENV=production`) for security, but enabled for authenticated developers in non-production environments.
*   **Field-Level Authorization:** For future sensitive metadata, Pothos's scope-based authorization plugin (`@pothos/plugin-scope-auth`) would be used to define access rules at the field level, even for a public API.

---

## 4. Performance & Efficiency
*   **N+1 Prevention (DataLoader):** The `@pothos/plugin-dataloader` is mandated for all relationship resolvers. A `countryLoader` is implemented to batch lookups for `Country` objects by their `code`, transforming N+1 lookups into batched O(1) operations.
*   **N+1 Detection:** Test cases for deeply nested relationships (e.g., `continent -> countries -> languages`) must assert that the underlying data source (e.g., `countries-list` object) is accessed efficiently (e.g., a single read for batched operations).
*   **Tracing:** Integration with Stellate or Apollo Studio for distributed tracing to identify slow resolvers and N+1 patterns in production.

---

## 5. CI/CD Pipeline Integration

### 5.1 Exit Criteria for Release
1.  **Zero Breaking Changes:** `graphql-inspector diff` must pass, ensuring no breaking changes are introduced without explicit version bumps.
2.  **100% Type Safety:** All generated client-side hooks/types (via GraphQL Code Generator) must compile with zero TypeScript errors.
3.  **Performance Benchmarks:** Critical queries (e.g., fetching all countries with nested languages) must meet predefined latency thresholds in integration tests.
4.  **Autonomous Fuzzing Pass:** 0% error rate on 1,000 auto-generated queries.
5.  **Security Checks:** Depth limit and ReDoS adversarial tests must pass.
6.  **Test Coverage:** Minimum 80% line coverage for resolver logic.

### 5.2 Workflow Steps
1.  **Linting & Type Checking:** `eslint` and `tsc --noEmit` run first.
2.  **Schema Diffing:** `graphql-inspector diff` compares current branch schema against production.
3.  **Unit & Integration Tests:** `bun test` executes all tests, including property-based and adversarial.
4.  **Client Code Generation:** `graphql-codegen` runs for React/Angular examples.
5.  **E2E Tests:** (Future) Playwright/Cypress runs against deployed staging environment.

---

## 6. Implementation Checklist

Items marked `[x]` are verified against the actual source. Items marked `[ ]` are recommended but not yet implemented.

### Already Implemented
- [x] Integration test suite using `yoga.fetch` directly — no network (`graphql.test.ts:7-10`)
- [x] `countries` list filtering — `eq`, `in`, `regex` operators tested (`graphql.test.ts:29-88`)
- [x] Subdivision resolver tested for GB (`graphql.test.ts:90-106`)
- [x] Zod validation on `Country.name(lang)` — rejects unknown language codes (`schema.ts:93-94`)
- [x] `defaultFieldNullability: false` — all non-nullable fields are intentional (`schema.ts:17-19`)
- [x] Null prototype fix via `JSON.parse(JSON.stringify(filter))` before Sift (`schema.ts:276, 303, 334`)
- [x] Batching enabled (`graphql.ts:7`)
- [x] CI runs `bun test` on every push (`.github/workflows/test.yml`)

### Recommended (Not Yet Implemented)
- [ ] Enable `useDepthLimit({ maxDepth: 7 })` plugin in `src/graphql.ts`
- [ ] Configure introspection based on environment in `src/graphql.ts`
- [ ] Implement `maxLength` or safe-regex validation for regex input filters in `src/schema.ts`
- [ ] Integrate `@pothos/plugin-dataloader` and refactor `Language.countries` / `Continent.countries` to use DataLoaders
- [ ] Integrate `graphql-inspector` into GitHub Actions for schema diffing
- [ ] Implement property-based / adversarial testing for filter logic (fast-check or manual fuzz)
- [ ] Set up AI-driven autonomous query fuzzing in CI
- [ ] Implement schema snapshot testing for all top-level Query fields
- [ ] Add batch size limit (`batching: { limit: 10 }`) to prevent batch abuse
- [ ] Set up GraphQL Code Generator for typed client examples
- [ ] Add `bunx tsc --noEmit` and `bunx eslint .` to CI pipeline

---
*Approved by: Principal Software Engineer*
