---
name: pr-description
description: Generates a pull request description by analyzing git commits, diffs, and changed files. Use when the user asks to write, create, or generate a PR description.
---

# PR Description Generator

## Instructions

### Step 1: Gather Git Context

Run the following commands in parallel to collect everything needed:

- `git log main..HEAD --oneline` — list all commits on this branch vs main
- `git diff main..HEAD --stat` — summary of files changed and line counts
- `git diff main..HEAD` — full diff of all changes

### Step 2: Read Changed Files

From the `--stat` output, identify the most significant changed files (prioritize non-test, non-config files). Read up to 5 of the most relevant files to understand the intent and context behind the changes — not just what changed, but why.

### Step 3: Infer the PR Structure

Based on the diff and file contents, determine the nature of the changes:

- **New feature** → use: Summary, What Changed, How to Test
- **Bug fix** → use: Summary, Root Cause, Fix, How to Test
- **Refactor** → use: Summary, What Changed, Why, Checklist
- **Docs/config only** → use: Summary, What Changed

Do not force a template that doesn't fit the change type.

### Step 4: Write the PR Description

Produce a clean, concise PR description in Markdown. Guidelines:

- **Title line**: one sentence, imperative mood (e.g., "Add depth limiting to prevent nested query abuse")
- **Summary**: 2–4 sentences on *what* changed and *why* — the business or engineering reason, not a restatement of the diff
- **What Changed**: bullet list of the key changes, grouped logically (not a file-by-file listing)
- **How to Test** (if applicable): numbered steps a reviewer can follow to verify the change works
- **Notes** (optional): breaking changes, follow-up TODOs, or anything a reviewer should be aware of

Keep it readable. Omit sections that have nothing meaningful to say.

### Step 5: Output

Print the PR description directly to the conversation so the user can copy it or request edits. Do not write it to a file unless the user asks.
