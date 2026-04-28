---
name: graphql-strategy-audit
description: Audits a GraphQL Test Strategy document against five mandatory QA requirements. Use when reviewing an existing TEST_STRATEGY.md for completeness, clarity, and risk coverage.
---

# GraphQL Strategy Auditor

## Instructions

Act as a Lead QA Engineer. Critically audit the project's `TEST_STRATEGY.md` against the five mandatory requirements below.

### Step 1: Read the Strategy Document

Read `TEST_STRATEGY.md` from the project root. If it does not exist, stop and tell the user to generate one first (e.g., using the `graphql-test-strategy` skill).

### Step 2: Ask About Custom Requirements

Before running the audit, ask the user:

> "Do you have any custom requirements to audit against, in addition to the five standard ones? (yes / no)"

- If the user says **yes**: ask them to share their custom requirements. Wait for their input. Append each custom requirement to the audit as additional numbered entries (6, 7, etc.), applying the same Met / Partially Met / Missed rating and one suggestion per entry.
- If the user says **no**: proceed directly to Step 3 with only the five standard requirements.

### Step 3: Audit Against Requirements

For each requirement, assign one of: **Met**, **Partially Met**, or **Missed**. Then provide one specific, actionable suggestion to improve the clarity or reasoning in the document.

**Requirement 1 — Inquiry & Discovery**
Did the strategy list the strategic questions asked *before* testing began? Look for a section that captures open questions about data integrity, resource constraints, consumer safety, or unknowns that shaped the testing approach.

- Met: Questions are explicitly listed with context about why they mattered.
- Partially Met: Some questions appear implicitly in the reasoning but are not called out as discovery questions.
- Missed: No discovery questions are documented at all.

**Requirement 2 — Prioritization**
Did the strategy identify 3–5 specific scenarios with a clear "Why" (risk-based reasoning)? Each scenario must name a concrete failure mode, not just a feature area.

- Met: 3–5 named scenarios, each with an explicit risk justification.
- Partially Met: Scenarios are listed but the "why" is vague or generic.
- Missed: No prioritized scenarios, or fewer than 3, or no risk reasoning.

**Requirement 3 — Scope & Trade-offs**
Did the strategy explicitly state what is NOT being tested and justify it based on time, cost, or feasibility constraints?

- Met: Out-of-scope items are named and each has a stated reason for deferral.
- Partially Met: Some items are excluded but the justification is missing or shallow.
- Missed: No explicit out-of-scope section.

**Requirement 4 — GraphQL-Specific Risks**
Did the strategy identify concerns unique to GraphQL? Look for: query depth attacks, N+1 resolver patterns, schema introspection leaks, breaking schema changes, nullability mismatches, or filter/input abuse surfaces.

- Met: At least three distinct GraphQL-specific risks are identified with concrete details (e.g., specific resolver names, field paths, or operators).
- Partially Met: GraphQL risks are mentioned but without specifics tied to the actual schema.
- Missed: Only generic API risks are listed with no GraphQL-specific concerns.

**Requirement 5 — AI Methodology**
Did the strategy document how AI was used in producing it? Look for: prompts used, AI output received, and how that output was refined or validated before inclusion.

- Met: Prompts, raw AI output, and refinement steps are all documented.
- Partially Met: AI involvement is acknowledged but the process is not described.
- Missed: No mention of AI methodology despite AI being used.

**Custom Requirements (if provided)**
For each custom requirement supplied by the user, apply the same three-tier rating (Met / Partially Met / Missed) and one specific suggestion. Number them starting at 6.

### Step 4: Write the Audit Report

Output a structured report in this format:

```
## GraphQL Strategy Audit Report

| Requirement | Status | Suggestion |
|---|---|---|
| 1. Inquiry & Discovery | [Met / Partially Met / Missed] | [one specific suggestion] |
| 2. Prioritization | [Met / Partially Met / Missed] | [one specific suggestion] |
| 3. Scope & Trade-offs | [Met / Partially Met / Missed] | [one specific suggestion] |
| 4. GraphQL-Specific Risks | [Met / Partially Met / Missed] | [one specific suggestion] |
| 5. AI Methodology | [Met / Partially Met / Missed] | [one specific suggestion] |
| 6. [Custom requirement name] | [Met / Partially Met / Missed] | [one specific suggestion] |

### Overall Assessment
[2–3 sentences on the document's strongest area and the single highest-priority gap to address.]
```

Omit row 6 (and beyond) if no custom requirements were provided.

Be specific: reference actual section names, line content, or missing items from the document rather than giving generic feedback.
