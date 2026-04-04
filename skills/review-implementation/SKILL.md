---
name: review-implementation
description: >
  Systematically reviews a codebase against a spec or task document, classifying
  every gap as a Bug (broken now), Minor (defensive concern), or Missing Feature
  (not built yet), and generates an actionable task doc for an agent to fix.
  Use this skill whenever the user says "review the implementation", "check if
  the spec was implemented", "compare code vs tasks", "did the agent implement
  everything?", or similar. Also trigger it when the user shares a .md tasks file
  and asks whether the code matches it. This skill is proactive — if the user has
  just received a PR or finished a feature sprint, suggest running it even if they
  didn't ask explicitly.
---

# Implementation Review Skill

You are performing a **spec-vs-code review**. Your job is to read the spec (PRD,
tasks doc, issue, or any written plan) and then audit the actual code, classifying
every gap you find and producing an actionable output document.

---

## Phase 0 — Orient

Before touching any file, understand the project structure:

1. Read `CLAUDE.md` (or equivalent project overview) if present.
2. Identify the tech stack (languages, frameworks, migration tool, test runner).
3. Note any conventions that affect correctness (e.g., Flyway versioning rules,
   active-record vs repository pattern, JWT role names, API error envelope shape).

Do this in one pass — you will need these facts to classify findings correctly.

---

## Phase 1 — Read the Spec

Read every section of the spec document carefully. For each described behaviour,
feature, or fix, create a mental checklist item. Pay attention to:

- **Exact field names and types** mentioned in the spec.
- **API endpoints and HTTP verbs** that should exist.
- **Database columns, indexes, or migrations** that should be added.
- **Validation rules and error codes** that must be enforced.
- **Frontend components, hooks, and URL parameters** that should be wired up.
- **Cross-cutting wiring**: a feature is only complete when *all layers* that
  touch it are implemented (DB → backend → API → frontend types → UI).

---

## Phase 2 — Audit the Code

For every checklist item from Phase 1, locate the relevant code and verify:

- Does the code exist at all?
- If it exists, does it match the spec (field names, logic, edge cases)?
- Are there runtime errors that would fire under normal use?

### Cross-cutting checks to always run

These catch whole classes of bugs that single-file reviews miss:

| Check | What to look for |
|---|---|
| **Frontend ↔ backend type sync** | TypeScript interfaces match the JSON the API actually returns |
| **Migration ↔ entity sync** | Every DB column in migrations exists in the ORM entity and vice versa |
| **Seeder ↔ schema sync** | Seed/demo data uses the same column names as the current schema |
| **Migration ordering** | New migrations don't use already-taken version numbers |
| **Config injection** | Framework config properties (e.g., `@ConfigProperty`) will fail at startup if the env var is missing — flag as a bug if no default is provided in production paths |
| **Null / Optional safety** | Nullable fields are handled before being dereferenced |
| **Error envelopes** | Error responses follow the project's documented shape |
| **Auth guards** | Authenticated routes actually check roles, not just the presence of a token |

---

## Phase 3 — Classify Every Finding

Use exactly these three categories:

### 🔴 Bug — Broken right now
Something that will produce an incorrect result, exception, or crash under normal
use. The feature may appear to work but produces wrong data, or it throws an
error that a user would encounter.

**Examples:** null pointer on a field that is sometimes absent, wrong SQL operator,
encoding bug in a string comparison, missing null-check before `.get()`.

### 🟡 Minor — Defensive concern
The code works today but is fragile, inconsistent, or likely to break as the
project grows. Prefer fixing these — defensive programming beats optimism.

**Examples:** missing `Optional` wrapping on an API key that could be unset,
no guard on a hook that only makes sense when a slug is present, a magic string
that should be a constant, an unhandled promise rejection.

### ❌ Missing Feature — Not built yet
The spec describes behaviour that simply does not exist in the codebase yet.

**Examples:** an entire API endpoint missing, a frontend filter that was specified
but never wired to the hook, a DB index that the spec calls for but no migration
adds.

> **Accuracy rule:** Never mark something as missing if it exists somewhere under
> a slightly different name or in a parent class. Always grep before concluding
> something is absent.

---

## Phase 4 — Produce the Output

### If the user wants a verbal summary

List findings grouped by category (Bugs first, then Minors, then Missing). For
each finding include:

- A one-line title
- The exact file + line (or function/method name) where the problem lives
- Why it is a problem (one or two sentences)
- A concrete fix suggestion

### If the user wants a task doc for an agent

Write a Markdown file (typically `docs/tasks-<feature>-fixes.md`) with this
structure:

```markdown
# Tasks: <Feature Name> — Fixes & Completion

## Context
<One paragraph explaining what was reviewed and what the spec said should exist.>

## Findings

### 🔴 Bugs (fix immediately)
#### Bug N: <title>
**File:** `path/to/file.ext`
**Problem:** <what goes wrong>
**Fix:** <exact change to make, with code snippet if helpful>

### 🟡 Minors (defensive improvements)
#### Minor N: <title>
**File:** `path/to/file.ext`
**Problem:** <why it's fragile>
**Fix:** <exact change to make>

### ❌ Missing Features (implement)
#### Feature N: <title>
**Files to create/edit:** `...`
**Spec says:** <quote or paraphrase from spec>
**Implementation:** <step-by-step, with code snippets for non-obvious parts>

## Verification Table

| # | Finding | Expected behaviour | How to verify |
|---|---|---|---|
| B1 | ... | ... | ... |
| M1 | ... | ... | ... |
| F1 | ... | ... | ... |

## Acceptance Criteria
- [ ] All bugs produce correct output under normal use
- [ ] All minors addressed or explicitly acknowledged
- [ ] All missing features pass their verification steps
- [ ] No new test failures introduced
```

---

## Phase 5 — Verify Your Own Findings

Before delivering output, re-read each finding and ask:

1. **Did I actually read the file, or am I guessing?** If you haven't read the
   file, read it now.
2. **Could this already be handled elsewhere?** Check parent classes, utility
   functions, middleware.
3. **Is my classification correct?** A missing feature is not a bug. A minor
   that will definitely fire is a bug.
4. **Is the fix I'm suggesting actually correct for this project's conventions?**
   (e.g., Flyway versioning, active-record pattern, JWT role strings)

Correct any errors you find. It is better to under-report than to send an agent
on a wild-goose chase fixing non-issues.

---

## Common Pitfalls

- **Stale task docs:** If the spec itself is a task list, some items may already
  be done. Always check the code before declaring something missing.
- **Migration version conflicts:** If a migration tool uses sequential numbers
  (like Flyway V1, V2, …), a new migration must use the *next available* number,
  not fill a gap in the sequence.
- **Framework startup failures vs runtime failures:** Some frameworks (Quarkus,
  Spring) validate config at startup. A missing required property crashes the
  whole app, not just one endpoint — this is always a Bug, not a Minor.
- **Type-vs-value confusion:** A TypeScript field marked optional (`?`) in the
  interface but required by the API (or vice versa) will silently misbehave.
  Always compare interface shape against actual API response shape.
- **Test mocks that lie:** If tests mock a hook or API and the mock doesn't
  match the real contract, the tests pass but the app can still be broken. Flag
  this as a Minor.

---

## Output Tone

- Be direct and specific. Vague findings ("this could be improved") are useless
  to an agent.
- Quote file paths and function names exactly as they appear in the code.
- For bugs, explain the failure mode — what the user would actually see go wrong.
- For missing features, give enough implementation detail that an agent can act
  without needing to re-read the entire spec.
