# Test Strategy Document
## GraphQL API — Production-Grade Test Strategy

**Role:** Senior QA Architect
**Scope:** Public read-only GraphQL API for countries, continents, and languages — live at `https://countries.trevorblades.com`
**Context:** Cloudflare Workers (128 MB memory, 10 ms CPU free / 30 ms paid), Pothos code-first schema builder, Sift.js filter engine, in-memory data from npm packages, Bun test runner
**Audience:** Engineering Leadership

---

## 0. Strategic Intent & Discovery

### 0.1 Discovery Questions

These questions were asked before any testing began. Answers shaped risk prioritization and scope decisions.

- **Data integrity**: When an upstream npm package (`countries-list`, `iso-3166-2`, `provinces`, `i18n-iso-countries`) releases a new version, how do we detect whether the raw data shape changed in a way that silently breaks a field resolver (e.g., `currencies` split on `,`, `subdivisions` filtered by type, `capital` coerced from empty string to null)?
  *Answer: No automated check exists today. Schema transformations are tested only through integration tests — and only 5 integration tests exist, covering `countries` filtering and one subdivision query. Upstream data shape changes are a blind spot. Identified as Priority Scenario 3.*

- **Resource constraints**: Given Cloudflare Workers cap CPU at 10 ms (free) / 30 ms (paid) and all filtering iterates over in-memory arrays per request, at what query complexity does the Worker hit its ceiling? How does deeply nested querying multiply that cost?
  *Answer: No depth limit or cost analysis is currently enforced. `Language.countries` and `Continent.countries` each perform a full scan of ~250 countries per resolved item — at depth 4, a `languages { countries { languages { countries } } }` query produces O(n³) resolver iterations. Identified as Priority Scenario 2.*

- **Consumer safety**: This API has no versioning and no advance-notice channel for breaking changes. How many consumers break silently if a field is removed or its nullability changes?
  *Answer: Currently undetectable before production — no schema diffing runs in CI, no schema snapshot is committed, and no `graphql-inspector` check exists. Any removed field or nullability flip silently breaks typed clients. Identified as Priority Scenario 4.*

### 0.2 Priority Scenarios (Risk-Based)

1. **ReDoS via `regex` filter** *(Severity: Critical)*
   `StringQueryOperatorInput.regex` (`schema.ts:216`) accepts arbitrary user strings passed directly to Sift's `$regex` with no length limit, timeout, or pattern validation. A crafted pattern such as `(a+)+$` causes catastrophic backtracking, exhausting the Worker's CPU budget and killing the isolate for all concurrent users — not just the attacker. No mitigation is currently in place.

2. **Unbounded nested traversal / depth bomb** *(Severity: High)*
   No depth limit is configured. `countries { languages { countries { languages { ... } } } }` is a valid query. Each hop performs a full O(n) scan of all countries, producing O(n³) iterations at depth 4. Blast radius: Worker CPU ceiling hit for the duration of the request, degrading the entire isolate.

3. **Silent data regression from upstream package updates** *(Severity: High)*
   All data originates from npm packages. Resolvers like `currencies` (`schema.ts:118`), `phones` (`schema.ts:107`), `capital` (`schema.ts:111`), and `subdivisions` (`schema.ts:163`) apply transformations (split, coerce, filter) that will silently produce wrong results if the upstream data format changes. No regression fixtures exist for these transformations.

4. **Breaking schema contract with no detection gate** *(Severity: High)*
   No `graphql-inspector diff` runs in CI. Removing a field, changing nullability, or renaming a type would reach production and break every consumer with a typed client. There is no schema snapshot committed to `main` and no CI gate preventing this.

5. **Batching abuse** *(Severity: Medium)*
   `batching: true` is configured (`graphql.ts:7`) with no per-batch size limit. A client can send an array of N operations in a single HTTP request, multiplying resolver load by N with no enforcement boundary other than the Worker's CPU ceiling.

### 0.3 Out of Scope & Trade-offs

- **Load / stress testing**: Deferred this cycle due to infrastructure setup time — a meaningful harness (k6, Artillery) against a Cloudflare Workers target requires edge deployment tooling not available now. Proxy signal: Cloudflare Analytics CPU time histograms. Revisit if batch abuse is observed.
- **Full i18n coverage**: Deferred due to diminishing returns — exhaustively asserting all 14 locales × 250 countries duplicates `i18n-iso-countries`' own test suite. Spot-checking 3–5 representative countries across 3 locales is sufficient to detect locale registration failures.
- **Mutation testing**: Out of scope — no mutations exist in the schema. Zero effort required until write operations are introduced.
- **Subscription testing**: Out of scope — no subscriptions exist in the schema.
- **E2E against live URL**: Deferred — integration tests via `yoga.fetch` already exercise the full stack without network calls. Live E2E tests add network flakiness without proportional coverage value in this cycle.

### 0.4 AI Methodology

**Model used:** Claude claude-sonnet-4-6

**Prompt used:**
> "Act as a Senior QA Architect. Create a comprehensive GraphQL Test Strategy Document for a production-grade supergraph. Include sections for: Testing Levels (Unit, Integration, E2E), Performance (load testing, N+1), Security (introspection, rate limiting, query complexity), Automation (CI/CD, schema validation, breaking change detection), and Tooling (Apollo GraphOS, Jest). Ensure the tone is technical and ready for an engineering leadership review."

**What the AI produced:**
A complete draft with all required sections, including specific file:line citations, the O(n³) complexity framing for nested resolvers, the ReDoS vector analysis, the `graphql-inspector` workflow for contract testing, and the DataLoader gap analysis.

**What was changed during human review:**
- All `[x]` checklist items were verified by re-reading source files before being marked implemented — the draft was accurate but treated as unverified until confirmed.
- ReDoS was elevated to Priority 1 with Severity: Critical (the draft placed it alongside other risks without ranking).
- Batching abuse was added as Priority 5 — the AI did not flag the absence of a batch size limit.
- Discovery questions were rewritten to include documented answers after each question.
- Out-of-scope justifications were expanded to include the proxy signal that replaces each deferred item.

---

## 1. Schema Governance & Type Safety

- **Approach**: Code-first with Pothos (`schema.ts:16`). The SDL is generated at runtime from `builder.toSchema()` — no hand-written `.graphql` file exists.
- **Nullability baseline**: `defaultFieldNullability: false` (`schema.ts:17-19`) — all fields non-nullable by default. Explicitly nullable fields: `capital` (`schema.ts:109`), `currency` (`schema.ts:113`), `State.code` (`schema.ts:199`), `Subdivision.emoji` (`schema.ts:67`), and single-item queries `continent`, `country`, `language` (`schema.ts:283`, `311`, `341`).
- **Breaking change detection**: Not implemented. Recommended: generate SDL from `builder.toSchema()`, commit as `schema.graphql` to `main`, and run `graphql-inspector diff` in CI before every merge. Exit criteria: zero breaking changes without explicit approval label.
- **Schema snapshot testing**: Not implemented. Recommended: commit baseline JSON response fixtures for all top-level queries and assert `deep.equal` in CI.
- **Schema linting**: `@graphql-eslint/eslint-plugin` is installed (`package.json:22`) but not run in CI — only `bun test` runs on push.
- **Client codegen**: Not used. Consumers query the live API directly with no generated types.

---

## 2. Testing Levels

### 2.1 Unit Tests — Resolver Logic

No isolated unit tests exist for resolver logic today. The following functions contain non-trivial transformation logic that must be tested independently of the full GraphQL stack:

| Function | File | Risk if untested |
|---|---|---|
| `currencies` split on `,` | `schema.ts:118` | Empty currency → `[""]` returned silently |
| `phones` split on `,` | `schema.ts:107` | Multi-dial-code countries (e.g., RU) produce wrong count |
| `capital` / `currency` empty-string-to-null coercion | `schema.ts:111,115` | `""` passes as non-null where `null` is expected |
| `subdivisions` filter (`type === "Country" or "Province"`) | `schema.ts:163` | Other subdivision types silently dropped |
| `Country.name(lang)` fallback when locale not registered | `schema.ts:98` | Unregistered locale silently falls back to English |
| `isValidContinentCode` / `isValidCountryCode` / `isValidLanguageCode` | `schema.ts:249-259` | Invalid code must return `null`, not throw |

**Tooling**: Bun test. No additional setup required.

### 2.2 Integration Tests — Schema-Driven

Tests execute via `buildHTTPExecutor` against `yoga.fetch` — no network, no external dependencies. The pattern (`graphql.test.ts:7-10`) is correct and should be extended.

**Currently covered** (`src/graphql.test.ts`):
- `countries` with `code: { in: [...] }` filter
- `countries` with `name: { regex: "^United" }` filter
- `countries` with `code: { eq: "US" }` filter
- `countries` with no filter (asserts count equals `countries-list` package size)
- `country(code: "GB").subdivisions` length assertion

**Critical gaps**:

| Scenario | Risk |
|---|---|
| `continent(code: "EU")` / `continent(code: "XX")` → `null` | Core query + nullable path — zero coverage |
| `language(code: "en")` / `language(code: "xx")` → `null` | Core query + nullable path — zero coverage |
| `country(code: "US") { name(lang: "de") }` | Localization path — zero coverage |
| `country(code: "US") { name(lang: "xx") }` | Zod validation error path — zero coverage |
| `country(code: "AQ") { capital }` → `null` | Nullable field returning null — zero coverage |
| `country(code: "US") { currencies phones }` | Split logic — zero coverage |
| `countries(filter: { code: { ne: "US" } })` | `ne` operator — zero coverage |
| `countries(filter: { code: { nin: ["US","CA"] } })` | `nin` operator — zero coverage |
| Batched HTTP request (two operations) | `batching: true` — zero coverage |

**Snapshot testing**: Bun test has no built-in snapshot support. Recommended: commit baseline JSON fixtures for `country(code: "US")` and `countries` (full list structure) and assert with `deepEqual`.

### 2.3 Contract Testing

No schema diffing is currently in place.

**Recommended workflow**:
1. Add a `generate-sdl` script that prints the SDL from `builder.toSchema()`.
2. Commit the output as `schema.graphql` to `main`.
3. Add a CI step: `graphql-inspector diff schema.graphql <(bun run generate-sdl)` before `bun test`.
4. Block PRs introducing breaking changes without an explicit approval label.

**Exit criteria**: Zero breaking schema changes reach `main` without maintainer sign-off.

### 2.4 End-to-End Tests — Critical Flows

Not warranted at this time. The API is read-only with no auth flows, and integration tests already exercise the full stack (schema → resolver → HTTP response). E2E tests against the live URL would add network flakiness with no proportional coverage value.

Revisit when: a UI consumer or authenticated flow is introduced.

---

## 3. Security

### 3.1 ReDoS — Critical Gap

`StringQueryOperatorInput.regex` (`schema.ts:216`) passes user strings directly to JavaScript's `RegExp` engine via Sift's `$regex`. No length limit, no pattern validation, no timeout.

**Recommended mitigations (priority order)**:
1. Validate with `safe-regex` or `vuln-regex-detector` inside the Zod validator at `schema.ts:93` before Sift executes.
2. If 1 is deferred, restrict `regex` to anchored patterns only (enforce `^` prefix).
3. If neither: rely on Cloudflare's 10/30 ms CPU ceiling as a backstop — but note this kills the isolate for all concurrent users.

### 3.2 Query Depth Limiting — Gap

No depth limit configured. `countries { languages { countries { languages { countries { ... } } } } }` is currently valid.

**Recommended**: `useDepthLimit({ maxDepth: 7 })` from `graphql-yoga`. Depth 7 covers all documented legitimate use cases while blocking recursive graph bombs.

### 3.3 Query Complexity Analysis — Gap

No cost analysis configured. A single query can fan out to O(n³) resolver calls with no budget enforcement.

**Recommended**: `graphql-query-complexity` with field costs: list fields (e.g., `countries`, `languages`) cost 10; scalar fields cost 1. Maximum budget: 1000. Block requests exceeding the budget before execution.

### 3.4 Introspection

GraphQL Yoga enables introspection by default in all environments. For a public API with a playground this is intentional. No change needed, but it must be a conscious decision — document it explicitly in `graphql.ts` as a comment.

### 3.5 Rate Limiting — Gap

No rate limiting is configured at the application or CDN layer. A client can issue unlimited requests per second, enabling sustained low-complexity flooding that stays under the per-request CPU ceiling but saturates the Worker's concurrent execution budget.

**Recommended enforcement layer**: Cloudflare WAF rate-limiting rule targeting `POST /graphql` — threshold: 300 requests/min per IP (covers legitimate high-frequency consumers while blocking scripted floods). No code change required; configured in the Cloudflare dashboard or via Wrangler's `wrangler.toml` `[triggers]` config.

**Test scenario**: Assert that requests beyond the threshold receive HTTP 429 responses (verifiable via integration test against a staging Worker with rate rules enabled, not in-memory).

### 3.6 Batching Abuse

`batching: true` (`graphql.ts:7`) with no per-batch size limit. A client can send N operations in one HTTP request.

**Recommended**: Set `batching: { limit: 10 }` to prevent trivial amplification.

### 3.7 Auth

Public, read-only API — no field-level authorization needed. If write operations are introduced, scope-based auth (`@pothos/plugin-scope-auth`) must be addressed before deployment.

---

## 4. Performance

### 4.1 N+1 Detection

No DataLoaders are used. Two resolvers scan all countries on every call:

| Resolver | Invocations per query | Worst-case iterations |
|---|---|---|
| `Continent.countries` | 1 per resolved Continent | 8 × 250 = 2,000 |
| `Language.countries` | 1 per resolved Language | 180+ × 250 = 45,000+ |

At current data size (all in-memory, ~250 countries), this is acceptable. **Detection strategy**: add an integration test asserting `languages { countries { code } }` completes in < 100 ms as a canary for O(n²) regressions.

### 4.2 DataLoader Strategy

No DataLoaders implemented. Recommended for `Language.countries` and `Continent.countries` if nested queries become a common consumer pattern. Batching window: single event-loop tick. Cache scope: per-request only (no cross-request caching in Workers).

### 4.3 Load Testing

Not implemented this cycle. Deferred — see Section 0.3. When implemented: use k6 targeting `countries(filter: { code: { in: [...] } })` with 100 VUs, assert p95 < 200 ms and zero CPU 1019 errors.

### 4.4 Cache Verification

No caching layer exists — all responses computed fresh per request. Cloudflare edge caching is not configured for this Worker. When/if added, verify via `CF-Cache-Status` response header assertions in integration tests.

### 4.5 Observability

No tracing configured. Cloudflare Analytics provides request counts, CPU time histograms, and error rates — sufficient for this use case without additional instrumentation. If resolver-level tracing is needed, integrate `graphql-yoga` tracing plugin or Apollo GraphOS.

---

## 4.6 Operational Health

### 4.6.1 Error Masking — Gap

GraphQL Yoga exposes `extensions.stacktrace` in error responses by default in development mode. If `NODE_ENV` is not explicitly set to `production` on the Cloudflare Worker, internal stack traces (file paths, line numbers, dependency versions) will leak to public consumers.

**Current state**: `wrangler.toml` contains no `[vars]` block and no `NODE_ENV` binding — the variable is unset at the Worker runtime level.

**Required action**:
1. Add `NODE_ENV = "production"` to `wrangler.toml` under `[vars]`.
2. Add an integration test asserting that a deliberately triggered error response (e.g., invalid `lang` argument) contains no `extensions.stacktrace` field:
   ```ts
   expect(result.errors[0].extensions).not.toHaveProperty("stacktrace");
   ```

### 4.6.2 Structured Audit Logs — Gap

No structured logging is configured. Cloudflare Workers supports `console.log` which appears in the Cloudflare dashboard log stream, but logs are unstructured, not queryable, and not persisted beyond the 7-day tail log window.

**Recommended**: Use Cloudflare Workers Logpush to export structured JSON logs to a storage sink (R2, Datadog, or Splunk). Minimum fields per request: `timestamp`, `operationName`, `queryDepth`, `responseTime`, `statusCode`, `cfRay`.

**Test scenario**: Assert that every GraphQL request writes a log entry containing at minimum `operationName` and `responseTime` fields — verifiable via integration test with a mock logging binding.

---

## 5. Automation & CI/CD

### 5.1 Exit Criteria for Release

| Gate | Target | Status |
|---|---|---|
| All integration tests pass | 0 failures | Enforced (`.github/workflows/test.yml`) |
| No breaking schema changes | 0 breaking diffs | Not implemented |
| Schema snapshot matches | 0 structural drift | Not implemented |
| Type check passes | 0 `tsc` errors | Not in CI |
| ESLint passes | 0 errors | Not in CI |
| ReDoS fuzz tests pass | 0 catastrophic patterns accepted | Not implemented |
| Batch size limit enforced | Requests > 10 ops rejected | Not implemented |

### 5.2 Workflow Steps

**Current** (`.github/workflows/test.yml`):
1. `actions/checkout@v3`
2. `oven-sh/setup-bun@v2`
3. `bun install`
4. `bun test`

**Recommended additions** (in execution order):
```yaml
- run: bunx tsc --noEmit                          # 1. Type safety gate
- run: bunx eslint .                               # 2. Lint gate
- run: bun run schema:diff                         # 3. Breaking change detection
- run: bun test                                    # 4. Unit & integration tests
- run: bun run test:snapshot                       # 5. Schema snapshot validation
- run: bun run test:security                       # 6. ReDoS fuzz + depth limit tests
```

### 5.3 Environment Matrix

Integration tests currently execute only against an in-memory `yoga.fetch` target. Regressions specific to the deployed Cloudflare Worker (wrangler compatibility date, environment variable differences, edge cold-start behavior, rate rule enforcement) are invisible until production.

| Environment | Target | Test Suite | Trigger |
|---|---|---|---|
| In-memory | `yoga.fetch` (no network) | Unit + all integration tests | Every PR push |
| Staging Worker | Deployed `wrangler deploy --env staging` | Smoke tests: 1 query per top-level resolver, error-masking assertion, batch size rejection | Pre-release gate |
| Production | `https://countries.trevorblades.com` | Read-only smoke tests: `country(code: "US") { name }`, `continents { code }` | Post-deploy verification only |

**Parity enforcement**: If a test passes in-memory but fails against the staging Worker, the staging failure is the source of truth. No release proceeds until both environments pass.

**Current status**: Only the in-memory row is implemented. Staging and production rows require a `wrangler deploy --env staging` environment to be configured in `wrangler.toml`.

---

## 6. Tooling Recommendations

| Concern | Tool | Status |
|---|---|---|
| Functional testing | Bun test | In use |
| Integration test executor | `@graphql-tools/executor-http` | In use |
| Schema validation / lint | `@graphql-eslint/eslint-plugin` | Installed, not in CI |
| Breaking change detection | `graphql-inspector` | Recommended |
| Monitoring & tracing | Cloudflare Analytics (basic) | In use |
| Resolver-level tracing | Apollo GraphOS | Recommended |
| Load testing | k6 | Recommended |
| Query depth limiting | `graphql-yoga` `useDepthLimit` | Recommended |
| Query complexity | `graphql-query-complexity` | Recommended |
| ReDoS prevention | `safe-regex` | Recommended |
| Snapshot testing | JSON fixtures + `deepEqual` | Recommended |

---

## 7. Implementation Checklist

### Already Implemented
- [x] Integration tests via `yoga.fetch` — no network (`graphql.test.ts:7-10`)
- [x] `countries` filtering — `eq`, `in`, `regex` operators tested (`graphql.test.ts:29-88`)
- [x] Subdivision resolver tested for GB (`graphql.test.ts:90-106`)
- [x] Zod validation on `Country.name(lang)` — rejects unknown language codes (`schema.ts:93-94`)
- [x] Zod validation errors surface as `VALIDATION_ERROR` extension code (`schema.ts:22-31`)
- [x] `defaultFieldNullability: false` — all non-nullable fields intentional (`schema.ts:17-19`)
- [x] Null prototype fix via `JSON.parse(JSON.stringify(filter))` before Sift (`schema.ts:276, 303, 334`)
- [x] Batching enabled (`graphql.ts:7`)
- [x] CI runs `bun test` on every push (`.github/workflows/test.yml`)

### Recommended (Not Yet Implemented)
- [ ] Unit tests for resolver logic (currency/phone splitting, empty-string coercion, subdivision filtering)
- [ ] Integration tests for `continent`, `language`, single-item `null` returns
- [ ] Integration tests for `ne` and `nin` filter operators
- [ ] Integration tests for `name(lang:)` localization — valid and invalid lang codes
- [ ] Integration tests for nullable fields: `capital`, `currency`, `State.code`
- [ ] Integration tests for `states`, `awsRegion`, `phones`, `currencies`
- [ ] Integration test asserting batched HTTP request is accepted
- [ ] ReDoS validation (`safe-regex`) in `StringQueryOperatorInput.regex` Zod validator
- [ ] Query depth limiting (`useDepthLimit({ maxDepth: 7 })`) in `src/graphql.ts`
- [ ] Query complexity budget (`graphql-query-complexity`) in `src/graphql.ts`
- [ ] Batch size limit (`batching: { limit: 10 }`) in `src/graphql.ts`
- [ ] Schema SDL snapshot committed and `graphql-inspector diff` in CI
- [ ] JSON response fixtures committed for snapshot regression testing
- [ ] `bunx tsc --noEmit` added to CI pipeline
- [ ] `bunx eslint .` added to CI pipeline
- [ ] N+1 canary test: `languages { countries { code } }` asserts < 100 ms
- [ ] Add `NODE_ENV = "production"` to `wrangler.toml` `[vars]`
- [ ] Integration test asserting no `extensions.stacktrace` in error responses
- [ ] Configure Cloudflare WAF rate-limiting rule: 300 req/min per IP on `POST /graphql`
- [ ] Configure Cloudflare Workers Logpush with structured JSON fields (`operationName`, `responseTime`, `cfRay`)
- [ ] Add `staging` environment to `wrangler.toml` and smoke test suite against deployed Worker
- [ ] Add post-deploy production smoke tests (`country(code: "US") { name }`, `continents { code }`)
