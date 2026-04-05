# Implementation Review Skill (Updated)

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

Before using the spec as ground truth, validate it. Specs go stale — they may
have been written before implementation started, or edited mid-sprint without
keeping up with the code.

Read the spec and flag any of the following:

| Issue | What to look for |
|---|---|
| **Already implemented** | Items described as "to do" that already exist in the code |
| **Stale references** | Version numbers, file paths, function names, or column names that don't match the current codebase |
| **Convention violations** | Instructions that contradict the project's own rules (e.g., wrong migration number, wrong error envelope shape) |
| **Contradictions** | Two sections of the spec that disagree with each other |
| **Ambiguity** | Requirements so vague they can't be verified ("improve performance", "handle errors better") |

Produce a brief **spec health verdict** before moving on:

- ✅ **Spec is clean** — proceed treating it as ground truth.
- ⚠️ **Spec has N stale/incorrect items** — list them, then treat the *code* as
  source of truth for those specific items and the spec as source of truth for
  the rest.

This verdict matters: a stale spec will cause you to file false bugs for things
the code already does correctly.

---

## Phase 2 — Read the Spec (build your checklist)

With a validated spec in hand, read every section carefully. For each described
behaviour, feature, or fix, create a mental checklist item. Pay attention to:

- **Exact field names and types** mentioned in the spec.
- **API endpoints and HTTP verbs** that should exist.
- **Database columns, indexes, or migrations** that should be added.
- **Validation rules and error codes** that must be enforced.
- **Frontend components, hooks, and URL parameters** that should be wired up.
- **Cross-cutting wiring**: a feature is only complete when *all layers* that
  touch it are implemented (DB → backend → API → frontend types → UI).

Skip any items you already flagged as stale in Phase 1.

---

## Phase 3 — Audit the Code

For every checklist item from Phase 2, locate the relevant code and verify:

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
| **Error envelopes** | Error responses follow the project's documented shape consistently across ALL resources |
| **Auth guards** | Authenticated routes actually check roles, not just the presence of a token |

---

## Phase 3b — Production Readiness Audit

Run this phase on every review, even if the feature being reviewed doesn't
touch these areas. Architectural bugs are invisible when you only read the
files the feature changed.

### Data integrity

| Check | What to look for |
|---|---|
| **FK cascade behavior** | `ON DELETE CASCADE` on FKs destroys child rows when the parent is deleted. Ask: *is that the right business rule?* For community data (reviews, posts) owned by one user but contributed by others, `RESTRICT` or `SET NULL` is usually correct. `CASCADE` is only right when children are truly owned by the parent. |
| **Orphan risk** | Can a delete leave rows with broken references? Is every FK either constrained or nullable? |
| **Aggregate consistency** | Denormalized counters / aggregates (e.g., `avg_rating`, `review_count`) — are they updated on every write path that changes the source data? Insert, update, and delete. |
| **Migration backfill safety** | Does a new `NOT NULL` column have a safe default for existing rows? A `DEFAULT '$$'` for a column that represents user intent backfills with fabricated data. |

### Authentication and authorization

| Check | What to look for |
|---|---|
| **Token expiry** | Does the token builder set an explicit `expiresAt`/`lifespan`? If not, tokens may never expire. Confirm via the framework's default behavior. |
| **Unguarded type parses on JWT claims** | `Long.parseLong(jwt.getSubject())`, `Integer.parseInt(claims.get("id"))` — these throw uncaught exceptions if the claim is missing or malformed. Wrap with try-catch and return 401. |
| **Role stored outside the token** | If the frontend stores `role` in localStorage separately from the token, it can be stale after a server-side role change. The role should be derived from the live token claim at hydration time. |
| **Rate limiting** | Auth endpoints (login, register, password reset) with no throttling are trivially brute-forceable. Flag if no reverse-proxy or application-layer rate limit exists. |
| **Dangerous seed defaults** | Demo users with known passwords seeded at startup — is seeding gated behind an env var? Is that var `false` by default in production? |

### Scalability and performance

| Check | What to look for |
|---|---|
| **Unbounded list endpoints** | Any `GET /collection` endpoint that returns all rows with no `LIMIT` or pagination params. One large result set can OOM the server or time out the client. |
| **Missing indexes** | For every column used in a `WHERE` clause or `ORDER BY`, confirm there is an index. Pay special attention to JSONB operators (`@>`, `->>`) which need GIN indexes, not B-tree. |
| **N+1 query patterns** | `FetchType.LAZY` associations accessed inside a loop (or a `.map()` over a result list) fire one SQL query per row. Verify that list endpoints use fetch joins or eager loading for associations accessed during serialization. |
| **In-memory filtering of large result sets** | Fetching all rows into the application layer before filtering (e.g., `openNow` filter) works at small scale but fails at production volume. Flag if the filtered set is unbounded. |

### Resilience and error handling

| Check | What to look for |
|---|---|
| **Check-then-act race conditions** | Read-then-write sequences (count + insert, find + persist) are not atomic. Concurrent requests can both pass the guard and collide at the DB constraint. The DB constraint is the real guard — the application must catch the constraint exception and return the appropriate HTTP status instead of a 500. |
| **Missing React Error Boundaries** | A runtime exception in any React component without an enclosing error boundary crashes the entire page. Every route-level component should have an error boundary. |
| **Input that produces invalid internal state** | E.g., a name field that, after normalization, produces an empty slug. The invalid state should be rejected at the boundary (validation layer) rather than failing deep inside a service. |

### Documentation and configuration drift

| Check | What to look for |
|---|---|
| **CLAUDE.md / README env var table** | Does every env var in the config file have a matching entry in the docs? Are documented defaults accurate? A mismatch means operators will misconfigure production. |
| **Commented-out migration gaps** | Missing version numbers in Flyway (V6, V7 absent when V5 and V8 exist) — are they intentional? Add a comment in the next migration or README if so, to prevent future confusion. |

---

## Phase 4 — Classify Every Finding

Use exactly these three categories:

### 🔴 Bug — Broken right now
Something that will produce an incorrect result, exception, or crash under normal
use. The feature may appear to work but produces wrong data, or it throws an
error that a user would encounter.

**Examples:** null pointer on a field that is sometimes absent, wrong SQL operator,
encoding bug in a string comparison, missing null-check before `.get()`.

Production readiness bugs (from Phase 3b) that qualify as 🔴:
- `ON DELETE CASCADE` that destroys community-owned data
- JWT tokens with no expiry
- Unbounded list endpoint on a table that will grow

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

## Phase 5 — Produce the Output

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

## Phase 6 — Verify Your Own Findings

Before delivering output, re-read each finding and ask:

1. **Did I actually read the file, or am I guessing?** If you haven't read the
   file, read it now.
2. **Could this already be handled elsewhere?** Check parent classes, utility
   functions, middleware.
3. **Is my classification correct?** A missing feature is not a bug. A minor
   that will definitely fire is a bug.
4. **Is the fix I'm suggesting actually correct for this project's conventions?**
   (e.g., Flyway versioning, active-record pattern, JWT role strings)
5. **For Phase 3b findings: did I apply domain judgment, not just pattern
   matching?** `ON DELETE CASCADE` is not always wrong — it depends on whether
   the child data belongs to the parent or to the community. Read the entity
   relationships and the business context before filing a data integrity bug.

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
- **Cascade delete and domain ownership:** `ON DELETE CASCADE` on a FK means
  "this child data is owned by the parent." If child data is instead
  *community-contributed* (reviews written by many users for one restaurant),
  cascade-deleting it when the restaurant owner leaves destroys other users'
  work. This requires domain judgment — the pattern match alone (`ON DELETE
  CASCADE` exists) is not enough to call it a bug.
- **Emergent bugs need multiple files:** Some bugs are only visible when two or
  more files are read together (JWT expiry requires auth resource + config file;
  cascade delete requires migration + entity + business domain). If Phase 3b
  findings seem thin, you probably didn't open the foundational files listed in
  Phase 0. Go back and open them.
