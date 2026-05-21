---
name: self-review
description: >
  Manual-only skill. Invoke ONLY when the user explicitly types the slash
  command `/self-review` (or the fully qualified `/vlad-claude:self-review`).
  Do not trigger on phrases like "review my changes", "audit my diff", or any
  other natural-language request — those should be handled directly without
  this skill. When invoked, audits the current local working tree (staged +
  unstaged + untracked) against the `coding-rules` skill and auto-fixes every
  violation found. This is for reviewing your own local work before pushing;
  it does NOT review remote pull requests.
---

Audit every locally modified file against the `coding-rules` skill and auto-fix what it finds. Runs in three phases. No PRD, no implementation — assume the code already exists locally and the user wants it scrubbed.

## Phase 1 — Collect the working tree

1. Run `git diff --name-only HEAD` to capture staged + unstaged changes. If the repo has no `HEAD` yet (fresh init), fall back to `git diff --name-only --cached` plus the untracked list.
2. Run `git ls-files --others --exclude-standard` to capture untracked new files. Include these — a newly written file is exactly the case the rules should police.
3. Union the two lists, deduplicate, and drop files that should not be audited:
   - generated files, lockfiles (`package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, etc.), `*.snap`
   - pure renames/moves with no content change (`git diff --name-status` reports them as `R100`)
   - binary assets (images, fonts, etc.)
4. If the resulting list is empty, tell the user "Working tree is clean — nothing to review" and stop. Do not invent work.

## Phase 2 — Audit in parallel

### Load the coding-rules content once

Sub-agents cannot rely on skill auto-loading, so the rule text must be inlined into their prompts. Before spawning:

1. Locate the `coding-rules` skill among the skills already advertised to you — it ships in the same plugin as this skill (typically a sibling directory). Resolve the path at runtime; do not hardcode it.
2. Read its `SKILL.md` in full and hold the text in memory for the rest of the run.
3. If `coding-rules` cannot be found, stop and tell the user — without the rules there is nothing to audit against.

### Shard the audit

Pick the sharding strategy based on the diff size. The goal is enough parallelism to be fast without drowning the harness in agents.

- **1–3 files**: one audit sub-agent per file.
- **4–10 files**: ~2–3 files per sub-agent, grouped by directory or feature so each agent sees related context together.
- **11+ files**: ~5 files per sub-agent. Cap total parallelism around 8 agents — beyond that, overhead outweighs speedup.

### Spawn audit agents

Spawn all audit sub-agents **in a single message with multiple Agent tool calls** so they run concurrently. Use `subagent_type: "general-purpose"`.

Each audit agent's prompt must contain:

1. The absolute paths of the files it owns.
2. The full text of the `coding-rules` skill — embed it inline. Do not pass a file path; the sub-agent has no guarantee it can resolve the skill on its own.
3. An instruction to read each file, walk every rule in the embedded text, and report violations as a structured list.
4. The required output format (below).

Required output format from each audit agent:

```
## <absolute/file/path>
- [<rule-name>] <line-number>: <one-line description of the violation> → <one-line fix>
- [<rule-name>] <line-number>: ...

## <absolute/file/path>
- (no violations)
```

Tell the audit agent: do not modify any files, do not run formatters, do not propose unrelated improvements. Only report violations of the rules in the embedded coding-rules text. Untracked files: audit the file's full content (everything is "new"). Modified files: focus on changed regions but flag clearly violating code in unchanged regions of the same file when it touches the rules — the user is reviewing the whole file now.

### Aggregate

Collect output from every audit agent into one consolidated list keyed by file path. If two agents disagree about a file (shouldn't happen with disjoint sharding, but check), the stricter reading wins.

If the consolidated list is empty across all files, skip Phase 3 and report success.

## Phase 3 — Auto-fix

Spawn a single fix sub-agent (`subagent_type: "general-purpose"`) with:

1. The consolidated violation list, verbatim.
2. The full text of the `coding-rules` skill inlined again — reuse what you loaded in Phase 2. The fix agent needs the rule definitions to apply fixes correctly.
3. Instructions: apply every listed fix, do not introduce new functionality, do not touch files not in the list, do not change behavior — only refactor to comply. If a fix would change observable behavior or conflicts with what the surrounding code clearly intends, leave it and report the conflict back instead of forcing the change.

After the fix agent returns, re-run `git diff --name-only HEAD` plus the untracked list and re-spawn audit agents **only on files the fix agent touched**. This catches violations the fix accidentally introduced. Cap this at one extra audit round — if violations still remain after that, report them to the user without looping further.

## Final summary

Print a short summary:

- Files reviewed: `<count>`
- Violations found: `<count>` across `<n>` files
- Violations fixed: `<count>`
- Violations remaining: `<count>` (list them inline if non-zero)

Do not commit, stage, or push. The user takes it from here.
