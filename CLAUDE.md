# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **GraphQL API for countries, continents, and languages** built with modern web standards. It serves real-world geographic data (countries, states/provinces, languages) through a GraphQL interface, optimized for runtime performance and minimal payload sizes.

### Architecture

- **Framework**: GraphQL Yoga (graphql-yoga) running on Cloudflare Workers via Wrangler
- **Schema Builder**: Pothos (code-first GraphQL builder — schema is defined in TypeScript, no SDL file)
- **Data Sources**:
  - `countries-list`: Base country/continent/language data
  - `iso-3166-2`: Detailed subdivision information
  - `provinces`: State/province mappings
  - `i18n-iso-countries`: Localized country names in 14+ languages
  - `country-to-aws-region`: AWS region proximity mapping
- **Query Filtering**: Sift.js with operators (`eq`, `ne`, `in`, `nin`, `regex`)
- **Validation**: Zod for type-safe input validation

### Key Files

- `/src/graphql.ts` - GraphQL Yoga server entry point; exports the `yoga` instance and `fetch` handler for Cloudflare Workers
- `/src/schema.ts` - Complete GraphQL schema definition with all types, resolvers, and filtering logic
- `/src/locales.ts` - Registers i18n locales (languages with >1% internet usage)
- `/src/graphql.test.ts` - Integration tests using GraphQL executor

## Development

### Setup

```bash
bun install
```

### Running Locally

```bash
bun run dev
```

Launches the GraphQL API at `http://localhost:8787/graphql` with a GraphQL playground.

### Linting & Formatting

```bash
bunx eslint .
bunx eslint . --fix
```

### Testing

```bash
# Run all tests
bun test

# Run a single test file
bun test src/graphql.test.ts

# Run tests matching a pattern (e.g., "filtered")
bun test --match "*filtered*"
```

Tests use Bun's native test runner. They execute against `yoga.fetch` directly (no network calls).

### Type Checking

```bash
bunx tsc --noEmit
```

## Schema Design Notes

### Type System

All schema types are non-nullable by default (`defaultFieldNullability: false`). Nullable fields are explicitly marked (e.g., `nullable: true` in `capital`, `currency`).

### Resolvers

- **Localization**: `Country.name(lang?: string)` accepts an optional language code to return translated country names via `i18n-iso-countries`
- **Nested Relations**: Most types include nested field resolvers:
  - `Continent.countries` → all countries in that continent
  - `Country.continent` → wraps continent code in Continent object
  - `Country.languages` → array of Language objects
  - `Country.states` → provinces via the `provinces` package
  - `Country.subdivisions` → ISO 3166-2 subdivisions with optional emoji flags
  - `Language.countries` → all countries speaking that language
- **Array Splitting**: `Country.currencies` and `Country.phones` split comma-separated strings into arrays (raw fields may contain multiple values)
- **AWS Regions**: `Country.awsRegion` resolves to the nearest AWS region
- **Deprecation**: `Country.states` is marked TODO for deprecation in favor of `Country.subdivisions` — do not add new callers of `states`

### Filtering

The filtering system uses Sift.js with custom operators. Example filters:
```graphql
query {
  countries(filter: { code: { in: ["US", "CA"] } }) { name }
  countries(filter: { currency: { eq: "USD" } }) { name }
  countries(filter: { name: { regex: "^United" } }) { name }
}
```

Filter inputs are normalized through `JSON.parse(JSON.stringify(filter))` to handle null prototype issues from the countries-list data source.

## Deployment

The application is deployed to **Cloudflare Workers** via automatic integration (no manual deploy action in CI/CD). The wrangler CLI is the build/deploy mechanism:

```bash
# Deploy to Cloudflare (requires authentication)
wrangler deploy
```

The live API runs at `https://countries.trevorblades.com`.

## CI/CD

GitHub Actions runs `bun test` on every push. No linting step in CI.

## Notes

- Batching is enabled in the Yoga config (`batching: true`)
- No GraphQL code generation (codegen) is used; the schema is hand-written with Pothos

