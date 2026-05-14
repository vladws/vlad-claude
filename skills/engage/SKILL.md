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

When implementation is done, capture the list of modified files with `git diff --name-only` (include both staged and unstaged). This list drives Phase 3.

## Phase 3 — Audit & auto-fix

Audit every modified file against the `coding-rules` skill in parallel, then auto-fix any violations. This phase always runs after implementation, even if you think the code is clean — the point is independent review.

### Locate the coding-rules content

Sub-agents you spawn cannot rely on auto-loading skills, so you must inline the full rule text into their prompts. Before spawning anything in Phase 3, load the `coding-rules` skill content once:

1. Find it by name in the available skills (it ships in the same plugin as this skill — typically as a sibling directory). Do not hardcode a path; resolve it at runtime by locating the `coding-rules` skill among the skills already advertised to you.
2. Read its `SKILL.md` file in full and hold the text in memory for the duration of Phase 3.
3. If you cannot locate `coding-rules`, stop and tell the user: Phase 3 cannot proceed without the rule definitions. Do not silently skip the audit.

### Sharding the audit

Pick the sharding strategy based on the diff size:

- **1–3 files**: one audit sub-agent per file. Maximum signal per agent.
- **4–10 files**: ~2–3 files per sub-agent, grouped by directory or feature when possible so each agent sees related context together.
- **11+ files**: ~5 files per sub-agent. Cap total parallelism around 8 agents — beyond that, the overhead outweighs the speedup.

Files that are only renamed/moved without content changes can be skipped. Generated files, lockfiles, and `*.snap` should also be skipped.

### Spawning audit agents

Spawn all audit sub-agents **in a single message with multiple Agent tool calls** so they run concurrently. Use `subagent_type: "general-purpose"`.

Each audit agent's prompt must contain:

1. The exact file paths it owns (absolute paths).
2. The full text of the `coding-rules` skill — embed the content you loaded in the "Locate the coding-rules content" step. Inline it; do not pass a file path. The sub-agent has no guarantee it can resolve the skill on its own.
3. An instruction to read each file, walk every rule in the embedded coding-rules text, and report violations as a structured list.
4. The required output format (below).

Required output format from each audit agent:

```
## <absolute/file/path>
- [<rule-name>] <line-number>: <one-line description of the violation> → <one-line fix>
- [<rule-name>] <line-number>: ...

## <absolute/file/path>
- (no violations)
```

Tell the audit agent: do not modify any files, do not run formatters, do not propose unrelated improvements. Only report violations of the rules in the embedded coding-rules skill. Empty diffs (file in list but no actual changes) → report "no violations".

### Aggregating violations

Collect the structured output from every audit agent. Build one consolidated violation list keyed by file path. If two agents disagree about a file (shouldn't happen with disjoint sharding, but check), the stricter reading wins.

If the consolidated list is empty across all files, skip the fix phase and report success.

### Spawning the fix agent

Spawn a single sub-agent (`subagent_type: "general-purpose"`) with:

1. The consolidated violation list (verbatim).
2. The full text of the `coding-rules` skill inlined again — reuse the content you loaded earlier; the fix agent needs the rule definitions to apply fixes correctly.
3. The instruction: apply every listed fix, do not introduce new functionality, do not touch files not in the list, do not modify behavior — only refactor to comply. If a fix conflicts with the PRD's intent, leave it and report the conflict back instead of forcing the change.

After the fix agent returns, re-run `git diff --name-only` and re-spawn audit agents **only on files the fix agent touched**. This catches violations the fix accidentally introduced. Cap this at one extra audit round — if violations still remain after that, report them to the user without looping further.

## Final summary

Print a short summary to the user:

- PRD implemented: `<path>`
- Files changed: `<count>`
- Violations found: `<count>` across `<n>` files
- Violations fixed: `<count>`
- Violations remaining: `<count>` (list them inline if non-zero)

Do not commit, stage, or push. The user takes it from here.
