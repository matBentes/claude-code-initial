# Review Prompt Templates

Two prompt templates for different review scopes. Use the right one — using the
PR template for a full audit will miss architectural concerns; using the full
audit template on a small PR is unnecessarily heavy.

---

## Template 1 — Feature PR Review

Use when reviewing a specific pull request or task doc implementation.

**Scope:** The changed files + their direct dependencies + cross-cutting wiring.
Does NOT audit the entire codebase for pre-existing issues outside the PR.

---

```
Read <task doc or PR description> and check if everything was implemented.

Follow the review-implementation skill guideline.

Scope:
- Verify every item in the task doc is implemented correctly
- Check cross-cutting wiring (DB → backend → API → frontend types → UI)
- Run the standard cross-cutting checks (type sync, migration sync, auth guards,
  error envelopes, null safety)
- Run Phase 3b Production Readiness checks ONLY for the files this PR touches —
  if a migration was added, check cascade behavior; if an auth endpoint was
  changed, check token expiry; if a new endpoint was added, check for pagination

Do NOT audit pre-existing code outside the PR scope for unrelated issues.

Produce a verbal summary grouped by 🔴 Bug / 🟡 Minor / ❌ Missing Feature.
If findings exist, offer to generate a task doc.
```

---

## Template 2 — Full Codebase / Architecture Review

Use periodically (before a release, after a major feature area is complete, or
when onboarding to an existing codebase). Not meant for every PR.

**Scope:** The entire codebase. Catches architectural, security, and data
integrity issues that are invisible in per-PR reviews.

---

```
Do a full production readiness review of this codebase.

Follow the review-implementation skill guideline, running ALL phases including
Phase 3b in full.

Mandatory file reads regardless of current feature scope:
- Every migration file (FK constraints, cascade behavior, missing indexes,
  backfill correctness)
- application.properties / config files (JWT expiry config, env var defaults,
  dangerous defaults)
- Auth resource + token builder (explicit expiry, claims, role encoding)
- Seed / demo data initializer (known credentials, startup guards)
- Every resource that parses JWT subject or claims as a primitive type

Review dimensions:
1. Spec compliance — does the code match the task docs / PRD
2. Cross-cutting correctness — type sync, error envelopes, auth guards
3. Data integrity — FK cascade rules, aggregate consistency, backfill safety
4. Auth security — token lifetime, unguarded parses, rate limiting, seed defaults
5. Scalability — unbounded queries, missing indexes, N+1 patterns, in-memory filtering
6. Resilience — check-then-act races, missing error boundaries, invalid state at input

For each finding:
- Cite the exact file and line
- Explain the failure mode (what a user or operator would actually experience)
- Classify as 🔴 Bug / 🟡 Minor / 🟡 SE Gap (architectural concern without a
  spec to violate)

After the verbal summary, generate a task doc at
docs/tasks-<scope>-fixes.md covering all actionable findings.
```

---

## When to use each

| Situation | Template |
|---|---|
| Reviewing a PR before merge | Template 1 |
| Checking a task doc was fully implemented | Template 1 |
| Pre-release audit | Template 2 |
| Onboarding to an existing codebase | Template 2 |
| After completing a major feature area | Template 2 |
| Something broke in production and you don't know why | Template 2 |

---

## Why two templates instead of one

A PR review and an architecture review have different goals:

**PR review** — was this specific change implemented correctly and safely?
Adding Phase 3b to every PR review is correct, but scoped to the changed files.
You don't need to re-audit the entire codebase's cascade behavior every time
someone adds a new filter to a search endpoint.

**Architecture review** — is this codebase safe to run at production scale?
This requires opening files that haven't changed recently (migrations, auth
config, seed data). A PR review will never open these unless the PR itself
touches them — which is exactly how the JWT expiry and cascade delete issues
were missed in the first two passes.

The two-template approach means:
- PRs stay fast and focused
- Architectural gaps are caught on a scheduled cadence, not left to chance
