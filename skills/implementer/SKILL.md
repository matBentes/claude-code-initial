# Implementer Skill

> **SESSION ROLE: IMPLEMENT ONLY**
> This skill is for **implement sessions**. It reads a task document (any
> `docs/tasks-*.md`) and executes it -- implementing features, applying fixes,
> writing migrations, wiring frontend to backend, whatever the plan describes.
> It does NOT review, re-classify, or question the plan -- it trusts the task
> document as ground truth and gets the work done.

---

## Phase 1 -- Find the Task Document

Look for a `docs/tasks-*.md` file. Priority order:

1. A file the user explicitly named in this conversation
2. The most recently modified `docs/tasks-*.md` in the repository
3. A task document pasted directly in the conversation

Read the full document before touching any file. If no task document exists,
ask the user which plan to implement and stop until they clarify.

---

## Phase 2 -- Orient Before Writing Code

Before making any change, spend one pass understanding the codebase:

1. Read `CLAUDE.md` (or equivalent) for project conventions.
2. Identify the tech stack so you use the right patterns (e.g., Panache active
   record vs repository, Flyway migration numbering, TypeScript strict mode).
3. For each task item, locate the files it touches -- read them before editing.
4. Note any shared utilities or constants relevant to the tasks -- reuse them
   instead of reinventing.

---

## Phase 3 -- Execute the Plan

Work through every item in the task document. Default order:

1. **Database / migrations first** -- schema changes must exist before the code
   that uses them.
2. **Backend** -- entities, services, resources, in dependency order.
3. **Frontend** -- types, hooks, components, wiring.
4. **Cross-cutting** -- error boundaries, config entries, documentation updates.

For each task item:

- Read the target file(s) if not already read this session.
- Implement exactly what the task describes -- no more, no less.
- If the task includes a code snippet, use it as the starting point.
- If a task item is already done in the codebase, skip it and note it in the summary.
- If a task item conflicts with the current codebase in a non-obvious way, implement
  the closest correct version and flag the discrepancy in the summary.

### Guardrails

- Do not add features not listed in the task document.
- Do not change test files unless the task document explicitly says to.
- Do not rename things not mentioned in the tasks -- renaming breaks callers.
- If a fix requires a new Flyway migration, use the next available version number.
- Do not refactor beyond what the task asks -- scope creep breaks sessions.

---

## Phase 4 -- Verify

After implementing all items, do a quick sanity pass:

- For bug fixes: confirm the error condition can no longer occur with the new code.
- For new features: confirm all layers are wired (DB to backend to API to frontend).
- For migrations: confirm the version number does not conflict with existing ones.
- For config additions: confirm a default value is set so the app starts without
  manual env var setup in development.

---

## Phase 5 -- Summary

Output a brief summary when done:

```
## Implementation Summary

### Done
- [file] what was implemented

### Skipped (already existed)
- [file] reason

### Flagged (discrepancy from task doc)
- [file] what differed and what was done instead

All done. X items implemented, Y skipped, Z flagged.
```
