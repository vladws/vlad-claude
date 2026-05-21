---
name: engage
description: >
  Implement a PRD end-to-end: pick a plan from `.ai/plans/`, build it, then run
  parallel sub-agents to audit every modified file against the coding-rules
  skill and auto-fix violations. Use only when the user explicitly invokes
  /engage or asks to "implement the PRD", "ship the plan", or "engage the
  plan". Do not use for ad-hoc coding tasks that lack a written PRD — those
  should be handled directly without this skill.
---

Implement a PRD, then enforce coding rules across everything that changed. Three phases, run in order.

## Phase 1 — Pick the PRD

1. List markdown files in `.ai/plans/` sorted by modification time (most recent first). If the directory does not exist, tell the user and stop.
2. Present the top candidates (with their first-line title) and ask the user which one to implement. If only one PRD exists and it was modified in the last hour, you can default to it and confirm in a single line.
3. Read the chosen PRD fully before writing any code. The PRD is the source of truth — do not invent scope that is not in it, and surface contradictions instead of guessing.

## Phase 2 — Implement

Implement the plan as described. Standard engineering rules apply:

- Read the surrounding code before changing it; match existing patterns.
- Keep the diff scoped to what the PRD calls for. Out-of-scope cleanups go in a separate change.
- If a decision in the PRD turns out to be wrong or impossible once you hit the code, stop and tell the user — do not silently substitute your own plan.

When implementation is done, hand off to Phase 3. The working tree now holds exactly the changes you made — the audit phase will pick them up from git directly.

## Phase 3 — Audit & auto-fix (delegated to `self-review`)

Audit every modified file against the `coding-rules` skill and auto-fix violations. This phase always runs after implementation, even if you think the code is clean — the point is independent review.

The full audit-and-fix workflow lives in the `self-review` skill. Do not re-implement it here.

1. Locate the `self-review` skill among the skills available to you — it ships in the same plugin as this skill (typically a sibling directory). Resolve the path at runtime; do not hardcode it.
2. Read `self-review/SKILL.md` in full and execute its Phases 1–3 verbatim. Treat its instructions as if the user had just typed `/self-review` — the self-review skill will derive the file list from git, load `coding-rules`, spawn the audit and fix sub-agents, and re-audit after fixing.
3. If you cannot locate the `self-review` skill, stop and tell the user: Phase 3 cannot proceed without it. Do not silently skip the audit.

Engage-specific note: when the review skill's fix agent encounters a violation whose fix would conflict with the PRD's intent, it should leave the code alone and report the conflict back. Make sure to surface those conflicts to the user in the final summary instead of forcing the change.

## Final summary

Once `self-review` finishes, print a short summary to the user that combines engage and review context:

- PRD implemented: `<path>`
- Files changed: `<count>`
- Violations found: `<count>` across `<n>` files (from review)
- Violations fixed: `<count>` (from review)
- Violations remaining: `<count>` (list them inline if non-zero)
- PRD conflicts (if any): `<list>`

Do not commit, stage, or push. The user takes it from here.
