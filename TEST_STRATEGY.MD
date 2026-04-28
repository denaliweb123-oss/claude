# Test Strategy Document
## GraphQL API Testing & AI-Augmented Quality Engineering

**Role:** Principal Software Engineer and Platform Architect
**Scope:** Public read-only GraphQL API at `https://countries.trevorblades.com`
**Backup API:** `https://rickandmortyapi.com/graphql` (Control Reference for platform health)
**Context:** Cloudflare Worker (128MB Memory Ceiling), Pothos Schema, Sift Filter Engine.

---

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
*   **Autonomous Query Fuzzing:** Implement a script that parses the generated SDL to produce 1,000+ random valid GraphQL queries, specifically targeting nested relationships to stress-test the Cloudflare Worker 128MB memory ceiling.
*   **AI-Generated Edge Cases:** Use Gemini/Claude to generate adversarial `filter` inputs (complex regex, nested logic) to stress-test the Sift engine.
*   **Synthetic Data Injection:** AI-driven generation of mock subdivision data to test the scalability of the `iso-3166-2` integration.
*   **Agentic Logic Discovery:** Deploy an LLM-based agent to "crawl" the graph and identify logical anomalies (e.g., checking if `Continent.countries` contains countries that claim to be in a different continent).

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
*   **Observability-Driven Testing (ODT):** Periodically ingest top-performing production query shapes into the integration test suite to ensure performance benchmarks reflect actual user behavior.

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
- [x] Enable `useDepthLimit` plugin in `src/graphql.ts`.
- [x] Configure `introspection` based on `NODE_ENV` in `src/graphql.ts`.
- [x] Implement `maxLength` validation for regex input filters in `src/schema.ts`.
- [x] Integrate `@pothos/plugin-dataloader` and refactor lookups to use DataLoaders.
- [ ] Integrate `graphql-inspector` into GitHub Actions for schema diffing.
- [x] Update `TEST_STRATEGY.md` to reflect all blueprint sections.
- [x] Implement property-based testing with fast-check for filter logic.
- [x] Add functional coverage for GraphQL Aliases and Fragments.
- [x] Hardened nullability for external field lookups (Partial Success resilience).
- [ ] Set up AI-driven autonomous query fuzzing in CI.
- [ ] Implement Schema Snapshot testing for all top-level Query fields.
- [ ] Integrate AI-based Semantic Validation for geographic data accuracy.
- [x] Implement Field-Level Authorization using Pothos scope-auth with API Key context.
- [x] Implement Query Cost Analysis to prevent fan-out attacks.
- [ ] Set up GraphQL Code Generator for React/Angular examples.

---
*Approved by: Principal Software Engineer*
