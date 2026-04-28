# Test Strategy Document
## GraphQL API Testing & AI-Augmented Quality Engineering

**Role:** Principal Software Engineer
**Scope:** Public GraphQL API for countries, continents, and languages — live at `https://countries.trevorblades.com`
**Context:** Cloudflare Workers (Wrangler), Pothos code-first schema builder, Sift.js filter engine, in-memory data sources (no database), Bun test runner

---

## 0. Strategic Intent & Discovery

### 0.1 Discovery Questions

These are the questions asked before any testing began. Answers shaped which risks were prioritized and what was left out of scope.

- **Data integrity**: When an upstream npm package (`countries-list`, `iso-3166-2`, `provinces`, `i18n-iso-countries`) releases a new version, how do we know whether the raw data shape changed in a way that silently breaks a field resolver (e.g., `currencies` split, `subdivisions` filter, empty-string `capital`)? *Answer: no automated check exists today — this is a gap.*
- **Resource constraints**: Given that Cloudflare Workers cap CPU time at 10 ms (free) / 30 ms (paid) and all filtering is done by iterating in-memory arrays on every request, at what query complexity does the Worker start hitting that ceiling? How does deeply nested querying (e.g., `languages { countries { languages { ... } } }`) multiply that cost? *Answer: no depth limit is currently enforced — quantified in Section 4.1.*
- **Consumer safety**: This API has no versioning and no advance-notice channel. How many consumers would break silently if a field were removed or its nullability changed? What is the current mechanism for detecting a breaking schema change before it reaches production? *Answer: none — identified as a critical gap in Section 2.3.*

### 0.2 Priority Scenarios (Risk-Based)

1. **ReDoS via `regex` filter** — `StringQueryOperatorInput.regex` accepts arbitrary user strings passed directly to Sift's `$regex` operator. A crafted pattern (e.g., `(a+)+$`) can cause catastrophic backtracking and exhaust the Cloudflare Worker's CPU budget, triggering a 1019 error for all users during that isolate's lifetime.

2. **Unbounded nested traversal** — The schema allows `countries { languages { countries { languages { ... } } } }` with no depth limit. Each hop iterates all countries, meaning depth-4 queries produce O(n³) resolver calls against the in-memory data.

3. **Breaking contract changes** — Fields like `Country.capital`, `Country.currency`, and `State.code` are explicitly nullable; `currencies` and `phones` split raw comma-separated strings. A patch to an upstream package (e.g., `countries-list`) that changes data format would silently break field resolvers.

4. **Missing test coverage for core resolvers** — 5 existing tests cover only `countries` list filtering. No tests exist for: `continent`/`language` queries, the `name(lang:)` localization argument, `states`/`subdivisions`/`awsRegion`, nullable field edge cases, or `ne`/`nin` filter operators.

### 0.3 Out of Scope & Trade-offs

- **Load/stress testing**: Deferred due to time constraints — setting up a meaningful load harness (k6, Artillery) against a Cloudflare Workers target requires infrastructure setup time not available in this cycle. Cloudflare's built-in CPU/request analytics will serve as a proxy signal for the first release. Revisit if batching abuse is observed in production.
- **Full i18n coverage**: Deferred due to diminishing returns — exhaustively asserting all 14 locales × 250 countries × all name fields would duplicate `i18n-iso-countries`' own test suite without adding confidence. Spot-checking 3–5 representative countries across 3 locales is sufficient to catch locale registration failures.
- **Mutation testing**: Out of scope because no mutations exist in the schema. Zero test effort required until write operations are introduced.
- **Subscription testing**: Out of scope because no subscriptions exist in the schema. Zero test effort required until real-time features are introduced.

### 0.4 AI Methodology — How This Document Was Produced

This strategy document was produced using Claude (claude-sonnet-4-6) as a drafting assistant. The process is documented here for auditability.

**Prompt used:**
> "Analyze this GraphQL API codebase and generate a comprehensive TEST_STRATEGY.md. Read the schema, existing tests, graphql.ts, locales.ts, CI config, wrangler.toml, and package.json. Identify risk areas including complexity hotspots, filter/input attack surface, public contract surface, nullability gaps, and missing coverage. Structure the document with discovery questions, risk-based priority scenarios, out-of-scope trade-offs, schema governance, testing pyramid, security/DoS risks, performance concerns, CI/CD gates, and an implementation checklist."

**What the AI produced:**
A complete draft covering all sections in this document, including specific file:line citations (e.g., `schema.ts:17-19` for nullability config), the N+1 worst-case iteration counts for `Language.countries` and `Continent.countries`, the Sift.js null-prototype issue explanation, and the five filter operator gap table.

**What was changed after review:**
- The checklist `[x]` items were verified by re-reading source files before being marked as implemented — the AI draft was accurate but treated as unverified until confirmed.
- The ReDoS risk was elevated to Priority 1 (the AI placed it alongside other risks without ranking); human judgment determined it was the most acute DoS vector given no CPU timeout exists per-regex.
- The "batching abuse" risk (Section 3.4) was added during human review — the AI did not flag the absence of a batch size limit.
- Discovery questions in Section 0.1 were initially written as statements; rewritten as explicit questions after an audit identified this as a gap.
- Out-of-scope justifications in Section 0.3 were initially thin; expanded with time-constraint rationale after the same audit.

---

## 1. Schema Governance & Type Safety

- **Approach**: Code-first with Pothos. The SDL is derived at runtime from `builder.toSchema()` — there is no hand-written `.graphql` file.
- **Nullability baseline**: `defaultFieldNullability: false` (`schema.ts:17-19`) means all fields are non-nullable by default. Nullable fields are explicitly opted in: `capital` (`schema.ts:109`), `currency` (`schema.ts:113`), `State.code` (`schema.ts:199`), `Subdivision.emoji` (`schema.ts:67`), and the `continent`/`country`/`language` single-item queries (`schema.ts:284`, `312`, `341`).
- **Breaking change detection**: Not currently implemented. Recommended addition: run `graphql-inspector diff` in CI against the schema snapshot committed to `main`, blocking merges that introduce breaking changes.
- **Schema linting**: `@graphql-eslint/eslint-plugin` is installed (`package.json:22`) but CI only runs `bun test` — ESLint is not enforced in the pipeline.
- **Client codegen**: Not used. Consumers query the live API directly.

---

## 2. The Testing Pyramid

### 2.1 Resolver Unit Tests

Currently there are no isolated unit tests for resolver logic. The following functions have non-trivial logic that warrants unit coverage independent of the full GraphQL stack:

| Function / Logic | File | Risk |
|---|---|---|
| `currencies` split on `,` | `schema.ts:118` | Countries with no currency return `[""]` |
| `phones` split on `,` | `schema.ts:107` | Multi-dial-code countries (e.g., RU) |
| `capital`/`currency` empty-string-to-null coercion | `schema.ts:111,115` | Silent data inconsistency |
| `subdivisions` filter (`type === "Country" or "Province"`) | `schema.ts:163` | Silently drops other subdivision types |
| `Country.name(lang)` fallback to `country.name` when locale missing | `schema.ts:98` | Locales not registered crash or silently fall back |
| `isValidContinentCode` / `isValidCountryCode` / `isValidLanguageCode` | `schema.ts:249-259` | Invalid codes must return `null`, not throw |

**Tooling**: Bun test (`bun test`). No additional setup required.

### 2.2 Integration Tests

Tests execute against `yoga.fetch` directly via `buildHTTPExecutor` — no network required. The executor pattern (`graphql.test.ts:7-10`) is correct and should be extended.

**Currently covered** (`src/graphql.test.ts`):
- `countries` with `code: { in: [...] }` filter
- `countries` with `name: { regex: "^United" }` filter
- `countries` with `code: { eq: "US" }` filter
- `countries` with no filter (returns all ~250)
- `country(code: "GB").subdivisions` — length assertion

**Gaps to fill**:

| Scenario | Why it matters |
|---|---|
| `continent(code: "EU")` happy path | Core query, zero test coverage |
| `continent(code: "XX")` → `null` | Nullable return path |
| `continents(filter: { code: { eq: "EU" } })` | Filter path for continents |
| `language(code: "en")` happy path | Core query, zero test coverage |
| `language(code: "xx")` → `null` | Nullable return path |
| `country(code: "US") { name(lang: "de") }` | Localization — registered locale |
| `country(code: "US") { name(lang: "xx") }` | Unregistered lang — Zod validation error |
| `country(code: "US") { capital currency }` | Non-null values for nullable fields |
| `country(code: "AQ") { capital }` → `null` | Antarctica has no capital |
| `country(code: "US") { currencies phones }` | Split logic for multi-value fields |
| `country(code: "US") { states { code } }` | States resolver, nullable `code` |
| `country(code: "US") { awsRegion }` | AWS region mapping |
| `countries(filter: { code: { ne: "US" } })` | `ne` operator — not tested |
| `countries(filter: { code: { nin: ["US","CA"] } })` | `nin` operator — not tested |
| `countries(filter: { currency: { eq: "USD" } })` | Filter on non-`code` field |
| `language(code: "en") { countries { code } }` | Reverse relation, scans all countries |
| Batched request (two operations in one HTTP body) | `batching: true` is configured |

### 2.2.1 Snapshot Testing

Structural snapshot tests are useful for catching silent regressions in data-heavy responses (e.g., full country list shape, subdivision structure for a specific country). Bun test does not have built-in snapshot support — consider committing baseline JSON fixtures and asserting with `deep.equal` against them.

### 2.3 Contract Testing

**Recommended workflow**:
1. Generate the SDL from `builder.toSchema()` and commit it as `schema.graphql` on `main`.
2. Add a CI step using `graphql-inspector diff schema.graphql <(bun run generate-sdl)` before `bun test`.
3. Block any PR that introduces a breaking change (removed field, changed nullability, renamed type) without an explicit approval label.

**Exit criteria**: Zero breaking schema changes merged without explicit maintainer approval.

### 2.4 End-to-End Tests

Not warranted at this time. The API is read-only, has no auth flows, and the integration tests already hit the full stack (schema → resolver → response). E2E tests against the live URL would add network flakiness without coverage value.

---

## 2.5 AI-Augmented Testing

- **Regex fuzzing**: Generate adversarial `regex` filter inputs (nested quantifiers, catastrophic backtracking patterns) and assert they either return an error or complete in < 5 ms. Example patterns: `(a+)+$`, `([a-z]+)*`, `^(([a-z])+.)+[A-Z]([a-z])+$`.
- **Filter input fuzzing**: Generate all combinations of `eq`/`ne`/`in`/`nin`/`regex` across `code`, `name`, `currency`, `continent` fields to surface Sift.js edge cases with null-prototype objects.
- **Synthetic locale coverage**: Use an LLM to generate country name assertions across all 14 registered locales for a representative set of 10 countries, then commit as regression fixtures.

---

## 3. Security & DoS Prevention

### 3.1 ReDoS (Current Gap)

The `regex` operator in `StringQueryOperatorInput` passes the user string directly to JavaScript's `RegExp` engine via Sift's `$regex`. There is no timeout, no pattern validation, and no allowlist.

**Recommended mitigations** (in priority order):
1. Validate regex patterns with a safe library (e.g., `safe-regex` or `vuln-regex-detector`) in the Zod validator before executing.
2. Alternatively, restrict the `regex` operator to anchored patterns only (must start with `^`).
3. If neither is implemented, rely on Cloudflare's 10/30 ms CPU limit as a hard backstop — but note this kills the isolate for all concurrent requests.

### 3.2 Query Depth & Complexity (Current Gap)

No depth limiting or complexity analysis is configured. The schema allows recursive traversal: `countries { languages { countries { languages { countries { ... } } } } }`.

**Recommended**: Add `useDepthLimit({ maxDepth: 7 })` from `graphql-yoga` or `graphql-depth-limit`. A depth of 7 covers all legitimate use cases while blocking pathological queries.

### 3.3 Introspection

GraphQL Yoga enables introspection by default. For a public API this is acceptable — introspection powers the playground. No change needed, but it should be a conscious decision rather than accidental.

### 3.4 Batching Abuse

`batching: true` is set (`graphql.ts:7`). A client can send an array of N queries in a single HTTP request, multiplying server load by N with no per-batch limit. Consider setting `batching: { limit: 10 }` to prevent trivial abuse.

### 3.5 Auth

This is a public, read-only API with no authentication. No field-level authorization is needed. If write operations are ever added, auth must be addressed before deployment.

---

## 4. Performance & Efficiency

### 4.1 N+1 Patterns

No DataLoaders are used. The following resolvers iterate over all ~250 countries on every call:

| Resolver | Called per... | Worst case |
|---|---|---|
| `Continent.countries` | Each resolved Continent | 8 continents x 250 scan = 2,000 iterations |
| `Language.countries` | Each resolved Language | 180+ languages x 250 scan = 45,000+ iterations |

For the current data size (all in-memory), this is acceptable. If the dataset grows or deep nesting becomes common, DataLoaders or result memoization should be introduced.

### 4.2 N+1 Detection in Tests

Add an integration test that queries `languages { countries { code } }` and asserts a response time under 100 ms. This acts as a canary for accidental O(n²) regressions.

### 4.3 Observability

No tracing or metrics are configured. Cloudflare's built-in analytics provide request counts and CPU time histograms — sufficient for this use case without additional instrumentation.

---

## 5. CI/CD Pipeline Integration

### 5.1 Exit Criteria for Release

| Gate | Target | Status |
|---|---|---|
| All integration tests pass | 0 failures | Enforced in CI |
| No breaking schema changes | 0 breaking diffs | Not yet implemented |
| Type check passes | 0 `tsc` errors | Not in CI |
| ESLint passes | 0 errors | Not in CI |
| Regex fuzz tests pass | 0 catastrophic patterns accepted | Not yet implemented |

### 5.2 Workflow Steps (Current)

From `.github/workflows/test.yml`:
1. `actions/checkout@v3`
2. `oven-sh/setup-bun@v2`
3. `bun install`
4. `bun test`

**Recommended additions**:
```yaml
- run: bunx tsc --noEmit          # type safety gate
- run: bunx eslint .               # lint gate
- run: bun run schema:diff         # schema contract gate (once SDL generation script exists)
```

---

## 6. Implementation Checklist

### Already Implemented
- [x] Integration test suite using `yoga.fetch` directly — no network (`graphql.test.ts:7-10`)
- [x] `countries` list filtering — `eq`, `in`, `regex` operators tested (`graphql.test.ts:29-88`)
- [x] Subdivision resolver tested for GB (`graphql.test.ts:90-106`)
- [x] Zod validation on `Country.name(lang)` — rejects unknown language codes (`schema.ts:93-94`)
- [x] Zod validation errors surface as `VALIDATION_ERROR` extension code (`schema.ts:22-31`)
- [x] `defaultFieldNullability: false` — all non-nullable fields are intentional (`schema.ts:17-19`)
- [x] Null prototype fix via `JSON.parse(JSON.stringify(filter))` before Sift (`schema.ts:276, 303, 334`)
- [x] Batching enabled (`graphql.ts:7`)
- [x] CI runs `bun test` on every push (`.github/workflows/test.yml`)

### Recommended (Not Yet Implemented)
- [ ] Unit tests for resolver logic (currency/phone splitting, empty-string coercion, subdivision filtering)
- [ ] Integration tests for `continent`, `language`, single-item queries, and `null` returns
- [ ] Integration tests for `ne` and `nin` filter operators
- [ ] Integration tests for `name(lang:)` localization — both valid and invalid lang codes
- [ ] Integration tests for nullable fields: `capital`, `currency`, `State.code`
- [ ] Integration tests for `states`, `awsRegion`, `phones`, `currencies`
- [ ] Regex safety validation (safe-regex or equivalent) before Sift execution
- [ ] Query depth limiting (`useDepthLimit` with `maxDepth: 7`)
- [ ] Batch size limit (`batching: { limit: 10 }`)
- [ ] Schema SDL snapshot committed and `graphql-inspector diff` in CI
- [ ] `bunx tsc --noEmit` in CI pipeline
- [ ] `bunx eslint .` in CI pipeline
