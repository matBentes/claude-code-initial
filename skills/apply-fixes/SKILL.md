# Apply Fixes Skill

> **SESSION ROLE: IMPLEMENT ONLY**
> This skill is for **implement sessions**. It takes a task document produced by
> `skills/review-implementation/SKILL.md` and applies every fix to the codebase.
> It does NOT re-review or re-classify — it trusts the task doc as ground truth.

---

## Phase 1 — Find the Task Document

Look for the most recent `docs/tasks-*.md` file in this conversation or in the
repository. If no task document exists, tell the user to run the review skill first
and stop.

Read the full task document before touching any file.

---

## Phase 2 — Apply All Fixes

Work through every finding in the document in order: Bugs first, then Minors,
then Missing Features.

For each fix:

1. Read the target file if you have not already in this session.
2. Apply the minimal change that resolves the finding as described.
3. If a finding is a false positive or clearly not applicable, skip it and note why.
4. Do not refactor beyond what the finding asks — scope creep breaks agent sessions.

Recommended order within each category:
- Data integrity bugs before auth bugs before performance bugs
- Larger structural changes before small cleanups

---

## Phase 3 — Verify Applied Fixes

After applying all changes, do a quick sanity pass:

- For each Bug fix: confirm the error condition described in the task doc can no
  longer occur with the new code.
- For each Missing Feature: confirm the new code matches what the spec described.
- For migrations: confirm the new version number does not conflict with existing ones.

---

## Phase 4 — Summary

Output a brief summary when done:

```
## Apply Fixes Summary

### Applied
- [file] what was fixed

### Skipped
- [file] reason it was skipped

All done. X fixes applied, Y skipped.
```

---

## Guardrails

- Do not add features not listed in the task document.
- Do not change test files unless the task document explicitly says to.
- Do not rename things that are not in the findings — renaming breaks callers.
- If a fix requires a new Flyway migration, use the next available version number.
- If a fix touches a shared utility used by many callers, note the impact in the summary.
