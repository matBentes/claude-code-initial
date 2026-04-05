# Implementation Review Skill

> **SESSION ROLE: REVIEW ONLY**
> This skill is for **review sessions**. It reads, audits, and reports findings.
> It does **NOT** edit any file, apply any fix, or write any code.
> Output is always a verbal report or a `docs/tasks-*.md` task document for an agent to act on later.
> If you feel the urge to fix something while running this skill — write it as a finding instead.
> To apply fixes, use the companion skill: `skills/implementer/SKILL.md`

---

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
4. **Always open these foundational files regardless of what the feature touches:**
   - Every migration file — FK constraints, cascade behavior, missing indexes
   - The app config file (`application.properties`, `.env`, `config.yml`) — JWT
     settings, secret defaults, feature flags
   - The auth resource + token builder — expiry, claims, role encoding
   - The seed / demo data initializer — dangerous defaults, known credentials

Do this in one pass — you will need these facts to classify findings correctly.
Skipping foundational files is the most common source of missed architectural bugs.

---

## Phase 1 — Review the Plan

Before using the spec as ground truth, validate it. Specs go stale.

Read the spec and flag any of the following:

| Issue | What to look for |
|---|---|
| **Already implemented** | Items described as "to do" that already exist in the code |
| **Stale references** | Version numbers, file paths, function names that do not match the codebase |
| **Convention violations** | Instructions that contradict the project's own rules |
| **Contradictions** | Two sections of the spec that disagree |
| **Ambiguity** | Requirements so vague they cannot be verified |

Produce a brief **spec health verdict** before moving on:

- OK **Spec is clean** — proceed treating it as ground truth.
- WARN **Spec has N stale/incorrect items** — list them, then treat the *code* as
  source of truth for those items.

---

## Phase 2 — Read the Spec (build your checklist)

For each described behaviour, feature, or fix, create a checklist item:

- Exact field names and types mentioned in the spec
- API endpoints and HTTP verbs that should exist
- Database columns, indexes, or migrations that should be added
- Validation rules and error codes that must be enforced
- Frontend components, hooks, and URL parameters that should be wired up
- Cross-cutting wiring: a feature is only complete when all layers are implemented
  (DB to backend to API to frontend types to UI)

Skip any items flagged as stale in Phase 1.

---

## Phase 3 — Audit the Code

For every checklist item from Phase 2, locate the relevant code and verify:

- Does the code exist at all?
- If it exists, does it match the spec (field names, logic, edge cases)?
- Are there runtime errors that would fire under normal use?

### Cross-cutting checks to always run

| Check | What to look for |
|---|---|
| **Frontend and backend type sync** | TypeScript interfaces match the JSON the API actually returns |
| **Migration and entity sync** | Every DB column in migrations exists in the ORM entity and vice versa |
| **Seeder and schema sync** | Seed data uses the same column names as the current schema |
| **Migration ordering** | New migrations do not use already-taken version numbers |
| **Config injection** | Config properties fail at startup if the env var is missing — flag if no default |
| **Null safety** | Nullable fields handled before being dereferenced |
| **Error envelopes** | Error responses follow the project documented shape across ALL resources |
| **Auth guards** | Authenticated routes actually check roles, not just the presence of a token |

---

## Phase 3b — Production Readiness Audit

Run this phase on every review, even if the feature does not touch these areas.
Architectural bugs are invisible when you only read the files the feature changed.

### Data integrity

| Check | What to look for |
|---|---|
| **FK cascade behavior** | ON DELETE CASCADE destroys child rows. For community data (reviews), RESTRICT is usually correct. |
| **Orphan risk** | Can a delete leave rows with broken references? |
| **Aggregate consistency** | Denormalized counters updated on every write path (insert, update, delete)? |
| **Migration backfill safety** | New NOT NULL column has a safe default for existing rows? |

### Authentication and authorization

| Check | What to look for |
|---|---|
| **Token expiry** | Does the token builder set an explicit expiresAt? If not, tokens may never expire. |
| **Unguarded type parses on JWT claims** | Long.parseLong without try-catch — wrap and return 401 on failure. |
| **Role stored outside the token** | Role in localStorage can be stale. Derive from the live token claim instead. |
| **Rate limiting** | Auth endpoints with no throttling are brute-forceable. |
| **Dangerous seed defaults** | Demo users seeded at startup gated behind an env var defaulting to false? |

### Scalability and performance

| Check | What to look for |
|---|---|
| **Unbounded list endpoints** | GET /collection returning all rows with no LIMIT or pagination |
| **Missing indexes** | Columns in WHERE or ORDER BY. JSONB operators need GIN indexes, not B-tree. |
| **N+1 query patterns** | LAZY associations accessed inside a loop or map() over a result list |
| **In-memory filtering** | Fetching all rows then filtering in application code — fails at production volume |

### Resilience and error handling

| Check | What to look for |
|---|---|
| **Check-then-act race conditions** | Read-then-write sequences are not atomic. Catch the DB constraint exception. |
| **Missing React Error Boundaries** | Route-level components without an error boundary crash the whole page |
| **Input producing invalid internal state** | A name that normalizes to an empty slug — reject at the boundary |

### Documentation and configuration drift

| Check | What to look for |
|---|---|
| **CLAUDE.md env var table** | Every env var in config has a matching entry in docs with accurate defaults |
| **Migration gaps** | Missing version numbers — intentional? Document it. |

---

## Phase 4 — Classify Every Finding

### BUG — Broken right now
Produces an incorrect result, exception, or crash under normal use.

### MINOR — Defensive concern
Works today but fragile. Likely to break as the project grows.

### MISSING — Not built yet
Spec describes behaviour that does not exist in the codebase.

Accuracy rule: Never mark something as missing if it exists under a slightly
different name. Always grep before concluding something is absent.

---

## Phase 5 — Produce the Output

**Reminder: output only. Do NOT edit any source file.**

### Verbal summary

List findings grouped by category (Bugs first, then Minors, then Missing):
- One-line title
- Exact file and line or function name
- Why it is a problem (one or two sentences)
- A concrete fix suggestion

### Task document for an agent

Write a Markdown file (typically docs/tasks-FEATURE-fixes.md) with this structure:

```
# Tasks: FEATURE NAME - Fixes & Completion

## Context
One paragraph explaining what was reviewed and what the spec said.

## Findings

### BUG (fix immediately)
#### Bug N: title
File: path/to/file.ext
Problem: what goes wrong
Fix: exact change to make, with code snippet if helpful

### MINOR (defensive improvements)
#### Minor N: title
File: path/to/file.ext
Problem: why it is fragile
Fix: exact change to make

### MISSING (implement)
#### Feature N: title
Files to create/edit: ...
Spec says: quote or paraphrase
Implementation: step-by-step with code snippets

## Verification Table

| # | Finding | Expected behaviour | How to verify |
|---|---|---|---|
| B1 | ... | ... | ... |

## Acceptance Criteria
- All bugs produce correct output under normal use
- All minors addressed or explicitly acknowledged
- All missing features pass their verification steps
- No new test failures introduced
```

---

## Phase 6 — Verify Your Own Findings

Before delivering output, re-read each finding and ask:

1. Did I actually read the file, or am I guessing? Read it now if not.
2. Could this already be handled elsewhere? Check parent classes and utilities.
3. Is my classification correct? A missing feature is not a bug.
4. Is the fix correct for this project conventions?
5. For Phase 3b findings: did I apply domain judgment? ON DELETE CASCADE is not
   always wrong — it depends on who owns the child data.

---

## Common Pitfalls

- Stale task docs: check the code before declaring something missing.
- Migration version conflicts: use the next available number, not fill a gap.
- Framework startup failures: a missing required config crashes the whole app — always a Bug.
- Type-vs-value confusion: compare interface shape against actual API response shape.
- Cascade delete and domain ownership: community-contributed data is not owned
  by the parent. Domain judgment required — pattern match alone is not enough.
- Emergent bugs need multiple files: JWT expiry requires auth resource plus config
  file. If Phase 3b findings seem thin, revisit Phase 0.
