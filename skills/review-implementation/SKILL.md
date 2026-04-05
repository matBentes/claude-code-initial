# Implementation Review Skill

> **SESSION ROLE: REVIEW ONLY**
> This skill is for **review sessions**. It reads, audits, and reports findings.
> It does **NOT** edit any file, apply any fix, or write any code.
> Output is always a verbal report or a `docs/tasks-*.md` task document for an agent to act on later.
> If you feel the urge to fix something while running this skill -- write it as a finding instead.
> To apply fixes, use the companion skill: `skills/implementer/SKILL.md`

---

You are performing a **full-spectrum review**. Your job is to read the spec and
the codebase and audit across six lenses: spec vs code, security, architecture,
simplification, test coverage, and completeness. Classify every gap and produce
an actionable output document.

---

## Phase 0 -- Orient

Before touching any file, understand the project structure:

1. Read `CLAUDE.md` (or equivalent project overview) if present.
2. Identify the tech stack (languages, frameworks, migration tool, test runner).
3. Note conventions that affect correctness (Flyway versioning, active-record vs
   repository, JWT role names, API error envelope shape).
4. **Always open these foundational files regardless of what the feature touches:**
   - Every migration file -- FK constraints, cascade behavior, missing indexes
   - The app config file (`application.properties`, `.env`, `config.yml`) -- JWT
     settings, secret defaults, feature flags
   - The auth resource + token builder -- expiry, claims, role encoding
   - The seed / demo data initializer -- dangerous defaults, known credentials

Do this in one pass. Skipping foundational files is the most common source of
missed architectural and security bugs.

---

## Phase 1 -- Validate the Plan

Before using the spec as ground truth, validate it. Specs go stale.

Read the spec and flag any of the following:

| Issue | What to look for |
|---|---|
| **Already implemented** | Items described as "to do" that already exist in the code |
| **Stale references** | Version numbers, file paths, function names that do not match the codebase |
| **Convention violations** | Instructions that contradict the project own rules |
| **Contradictions** | Two sections of the spec that disagree |
| **Ambiguity** | Requirements so vague they cannot be verified |

Produce a **spec health verdict** before moving on:

- OK **Spec is clean** -- proceed treating it as ground truth.
- WARN **Spec has N stale/incorrect items** -- list them, then treat the *code*
  as source of truth for those items.

---

## Phase 2 -- Build the Checklist

For each described behaviour, feature, or fix, create a checklist item:

- Exact field names and types mentioned in the spec
- API endpoints and HTTP verbs that should exist
- Database columns, indexes, or migrations that should be added
- Validation rules and error codes that must be enforced
- Frontend components, hooks, and URL parameters that should be wired up
- Cross-cutting wiring: a feature is only complete when all layers are
  implemented (DB to backend to API to frontend types to UI)

Skip items flagged as stale in Phase 1.

---

## Phase 3 -- Six-Lens Audit

Run all six lenses. Launch them as parallel agents when possible, each receiving
the full diff and the foundational files list from Phase 0.

---

### Lens 1: Spec vs Code

For every checklist item from Phase 2, locate the relevant code and verify:

- Does the code exist at all?
- If it exists, does it match the spec (field names, logic, edge cases)?
- Are there runtime errors that would fire under normal use?

**Cross-cutting checks:**

| Check | What to look for |
|---|---|
| Frontend and backend type sync | TypeScript interfaces match the JSON the API actually returns |
| Migration and entity sync | Every DB column in migrations exists in the ORM entity and vice versa |
| Seeder and schema sync | Seed data uses the same column names as the current schema |
| Migration ordering | New migrations do not use already-taken version numbers |
| Config injection | Config properties fail at startup if the env var is missing -- flag if no default |
| Null safety | Nullable fields handled before being dereferenced |
| Error envelopes | Error responses follow the project documented shape across ALL resources |
| Auth guards | Authenticated routes actually check roles, not just the presence of a token |

---

### Lens 2: Security

| Check | What to look for |
|---|---|
| **Token expiry** | Does the token builder set an explicit `expiresAt`? If not, tokens may never expire. |
| **Unguarded JWT claim parses** | `Long.parseLong(jwt.getSubject())` without try-catch -- wrap and return 401 on failure. |
| **Role stored outside the token** | Role in localStorage can be stale after a server-side change. Derive from the live token claim. |
| **Rate limiting** | Auth endpoints with no throttling are brute-forceable. |
| **Dangerous seed defaults** | Demo users seeded at startup -- gated behind an env var defaulting to false in production? |
| **FK cascade behavior** | ON DELETE CASCADE on community-contributed data (reviews, posts) destroys other users work when the parent is deleted. RESTRICT is usually correct. |
| **Orphan risk** | Can a delete leave rows with broken references? |
| **Input boundary validation** | Is user input validated at the entry point before reaching service or DB layer? |
| **Mass assignment** | Are request DTOs explicit about which fields are writable, or does the endpoint accept and bind arbitrary fields? |
| **Insecure direct object reference** | Does the endpoint verify the caller owns the resource before acting on it? |

---

### Lens 3: Architecture and Design

| Check | What to look for |
|---|---|
| **Shallow modules** | Classes or functions that expose more than they hide -- a method that requires the caller to know internal state to use correctly. |
| **Deep coupling** | Two modules that import each other or share mutable state in a way that makes them impossible to test independently. |
| **Wrong layer of abstraction** | Business logic in a controller/resource, or DB queries in a service that should be in a repository/entity. |
| **Abstraction boundary violations** | A new class bypassing an existing service to call the DB directly, or a frontend component importing from another component internal directory. |
| **God classes / bloat** | A single class growing responsibilities it should delegate. |
| **Duplicated domain logic** | The same validation rule or calculation expressed in two places that could diverge. |
| **Framework misuse** | Using a framework feature against its intended grain (e.g., triggering side effects in a Panache entity constructor, or doing blocking I/O in a reactive handler). |

---

### Lens 4: Simplification

| Check | What to look for |
|---|---|
| **Code reuse** | New functions that duplicate existing utilities. Search adjacent files and shared modules before flagging. |
| **Redundant state** | State that duplicates existing state, or cached values that could be derived on the fly. |
| **Copy-paste with slight variation** | Near-duplicate blocks that should be unified with a shared abstraction. |
| **Stringly-typed code** | Raw strings where constants, enums, or branded types already exist. |
| **N+1 patterns** | LAZY associations accessed inside a loop or `.map()` over a result list -- each fires a separate SQL query. |
| **In-memory filtering of large sets** | Fetching all rows then filtering in application code -- fails at production volume. |
| **Missed concurrency** | Independent operations run sequentially that could run in parallel. |
| **Unnecessary comments** | Comments explaining WHAT the code does -- keep only non-obvious WHY. |

---

### Lens 5: Test Coverage

| Check | What to look for |
|---|---|
| **Happy path only** | Tests cover the success case but not auth failure, invalid input, conflict, or not-found. |
| **Missing edge cases** | Boundary values, empty collections, null optional fields, concurrent writes. |
| **Mocks that lie** | A test mock that does not match the real contract -- tests pass but the app can still be broken. |
| **No integration coverage** | New endpoints with only unit tests but no test that exercises the full DB-to-HTTP stack. |
| **Untested error paths** | A `catch` block or error response branch that is never exercised by any test. |
| **Coverage gaps on new code** | Any new file or method with zero test coverage. |

---

### Lens 6: Completeness

**Migration safety:**

| Check | What to look for |
|---|---|
| **NOT NULL without default** | A new NOT NULL column with no DEFAULT forces every existing row to provide a value at migration time -- this fails on a non-empty table. |
| **Lock-heavy operations** | Adding an index or constraint on a large table locks writes. Flag if there is no `CONCURRENTLY` or equivalent. |
| **Backfill correctness** | A DEFAULT value that represents user intent (e.g., a fabricated slug) is silently wrong data. |
| **Irreversibility** | Column drops, renames, or type changes that cannot be rolled back without data loss. |

**Frontend completeness:**

| Check | What to look for |
|---|---|
| **Loading states** | Every async operation has a visible loading indicator. |
| **Error states** | Every async operation has a user-facing error message, not just a console log. |
| **Empty states** | List views handle the zero-results case with a message, not a blank page. |
| **Error boundaries** | Route-level components are wrapped in an error boundary so one crash does not take down the whole page. |

**Observability:**

| Check | What to look for |
|---|---|
| **Silent catch blocks** | `catch (e) {}` or `catch (e) { return null; }` with no logging -- failures become invisible in production. |
| **Missing structured logging** | New service-layer operations with no log entry on failure, making debugging from production logs impossible. |
| **CLAUDE.md / README drift** | Every new env var, API endpoint, or credential in the code has a matching entry in the docs. |

---

## Phase 4 -- Classify Every Finding

### BUG -- Broken right now
Produces an incorrect result, exception, or crash under normal use.

### MINOR -- Defensive concern
Works today but fragile. Likely to break as the project grows.

### MISSING -- Not built yet
Spec describes behaviour that does not exist in the codebase.

> **Accuracy rule:** Never mark something as missing if it exists under a slightly
> different name. Always grep before concluding something is absent.

---

## Phase 5 -- Produce the Output

**Reminder: output only. Do NOT edit any source file.**

### Verbal summary

List findings grouped by lens, then by severity within each lens:
- One-line title
- Exact file and line or function name
- Why it is a problem (one or two sentences)
- A concrete fix suggestion

### Task document for an agent

Write a Markdown file (`docs/tasks-<feature>-fixes.md`):

```
# Tasks: FEATURE NAME -- Fixes & Completion

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

| # | Finding | Lens | Expected behaviour | How to verify |
|---|---|---|---|---|
| B1 | ... | Security | ... | ... |

## Acceptance Criteria
- All bugs produce correct output under normal use
- All minors addressed or explicitly acknowledged
- All missing features pass their verification steps
- No new test failures introduced
```

---

## Phase 6 -- Verify Your Own Findings

Before delivering output, re-read each finding and ask:

1. Did I actually read the file, or am I guessing? Read it now if not.
2. Could this already be handled elsewhere? Check parent classes and utilities.
3. Is my classification correct? A missing feature is not a bug.
4. Is the fix correct for this project conventions?
5. For Security findings: did I check both the code and the config? Many
   security issues are only visible when two files are read together.
6. For Architecture findings: is this actually a problem for this codebase size
   and team, or am I over-engineering?
7. For Simplification findings: would the suggested abstraction actually be
   simpler, or just different?

---

## Common Pitfalls

- **Stale task docs:** Check the code before declaring something missing.
- **Migration version conflicts:** Use the next available number, not fill a gap.
- **Framework startup failures:** A missing required config crashes the whole app -- always a Bug.
- **Type-vs-value confusion:** Compare interface shape against actual API response shape.
- **Cascade delete and domain ownership:** Community-contributed data is not owned
  by the parent. Domain judgment required -- pattern match alone is not enough.
- **Emergent bugs need multiple files:** JWT expiry requires auth resource plus
  config file. If Lens 2 findings seem thin, revisit Phase 0.
- **Encoding-vs-content confusion:** When quoting code as evidence, copy from the
  raw file -- not from a rendered view or terminal that may re-encode characters.
  Unicode dashes displayed as mojibake are a display problem, not a code problem.
  Grep the raw bytes before filing a character-handling finding.
- **Database NULL sort defaults:** PostgreSQL ASC ordering places NULLs LAST by
  default. Do not file a NULLS LAST missing bug without confirming the target
  database. Downgrade to Minor (implicit reliance on engine default) if the
  codebase targets PostgreSQL exclusively.
- **Over-engineering architecture findings:** Flag coupling that causes real
  testability or maintainability pain, not every deviation from a textbook pattern.
