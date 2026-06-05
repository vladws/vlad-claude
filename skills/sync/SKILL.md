---
name: sync
description: >
  Sync web and mobile implementations of the same feature, maintaining a
  verbatim 1-to-1 port between a source and destination directory. Use when
  the user invokes /sync, asks to "sync web to mobile", "sync mobile to web",
  "port this feature to the other platform", "propagate changes to mobile/web",
  or says something like "keep web and mobile in sync". Also trigger when the
  user asks to set up a sync relationship between two feature folders, or when
  they've changed the source flow and want the destination updated. Do NOT
  trigger for general "sync" meanings like git sync, database sync, or
  unrelated file operations.
---

Sync a source feature flow to a destination platform, maintaining a verbatim
1-to-1 port. The sync direction is **unidirectional** — established once at
setup and recorded in CLAUDE.md. Source → destination is fixed; to reverse the
direction, the user must explicitly re-run setup with swapped arguments.

Runs in three phases: identify flows, set up documentation, then perform the
sync.

---

## Phase 1 — Identify source and destination

### If the user provided explicit folder paths

Accept them. Then confirm before proceeding:

> "Syncing from `<source>` → `<destination>`. Is that correct?"

Do not proceed until the user confirms.

### If the user was vague (e.g. "sync the e-transfer flow")

Search the codebase for folders that plausibly match the described feature.
Look in common locations: `src/features/`, `src/flows/`, `src/screens/`,
`src/pages/`, `apps/`, `packages/` — and any location where similar synced
pairs already exist (check for `CLAUDE.md` files with sync metadata). Present
the candidates with their full paths:

> "I found these candidates:
>  - Source (web): `src/features/e-transfer/web/`
>  - Destination (mobile): `src/features/e-transfer/mobile/`
>
> Is this correct, or should I use different paths?"

Do not proceed until the user confirms the exact source and destination paths.
Never assume — a wrong sync target can corrupt a platform's codebase.

---

## Phase 2 — Set up CLAUDE.md documentation

CLAUDE.md files are the long-lived contracts that guide agents maintaining this
sync. They must exist in both source and destination directories, and must be
kept accurate on every run.

### Check existing CLAUDE.md files

1. Check if `<source>/CLAUDE.md` and `<destination>/CLAUDE.md` exist.
2. If either exists, read it. Look for a `## Sync` section.
3. If a sync section already exists, verify the paths and exceptions are still
   accurate. Update anything that has changed. Confirm any changes with the
   user before writing.
4. If no sync section exists (or the file is new), create one using the
   templates below.

### Record explicit exclusions from the invocation

If the user mentioned any exclusions when invoking the skill (e.g. "exclude
the interac section"), record those in the CLAUDE.md templates below. Do not
ask for additional exclusions upfront — platform conflicts are discovered and
resolved during Phase 3.

### CLAUDE.md template — source directory

```markdown
## Sync

This directory is the **source of truth** for the `<feature name>` flow.

**Synced to:** `<relative path to destination>`

For approved exceptions and exclusions, see the destination CLAUDE.md at
`<relative path to destination>/CLAUDE.md`.

### Rules for agents working here
- After modifying any logic (components, hooks, utilities, types) in this
  directory, you MUST run `/sync` to propagate changes to the destination.
  If the `sync` skill is not available, notify the user and ask them to check
  `CODEOWNERS` for this directory to contact the responsible team.
- Do not let logic diverge between source and destination — the sync must be
  maintained verbatim. Exceptions are listed in the destination CLAUDE.md.
```

### CLAUDE.md template — destination directory

```markdown
## Sync

This directory is a **synced port** of `<relative path to source>`.

**Synced from:** `<relative path to source>`

### CRITICAL — do not modify this directory directly
Agents and contributors must NOT modify files in this directory for logic
changes. The only correct workflow is:
1. Modify the source at `<source path>`
2. Run `/sync` to propagate changes here (requires the `sync` skill)
3. Only make platform-specific adjustments (listed in "Approved exceptions"
   below) directly in this directory

If you are about to change logic here (component behavior, hooks, utilities,
business rules), **stop** and make the change in the source instead, then run
`/sync`. If the `sync` skill is not available in your environment, do not
attempt to port the changes manually — notify the user and ask them to check
`CODEOWNERS` for the source directory to contact the responsible team.

### Sync contract
This directory is a verbatim 1-to-1 port of the source. Component names, file
and folder names, directory hierarchy, logic, utility functions, variable
names, and code comments all match the source exactly.

**Approved exceptions (confirmed by user during sync runs):**
<list each exception as: "source uses X → destination uses Y (file reference where pattern was found)"; or "None" if none yet>

**Explicit exclusions (user-confirmed):**
<list each user-confirmed exclusion, or "None" if none>

**Platform-specific files owned by this directory (not synced):**
<list any files that exist only in this platform and are never overwritten
by sync — e.g. platform navigation wrappers, wallet integrations>
```

Write the filled-in versions of these templates to the respective CLAUDE.md
files. If a CLAUDE.md already has other content (not sync-related), preserve it
and append or replace only the `## Sync` section.

---

## Phase 3 — Perform the sync

Walk the source directory and produce a verbatim port in the destination.

### File-level mapping

For every file in source:
1. Compute the corresponding destination path by replacing the source root with
   the destination root (preserving all intermediate directories).
2. Determine if the file is excluded (matches an explicit exclusion from
   CLAUDE.md or is a known platform-only config file). If excluded, skip it
   and note it in the sync report.
3. If not excluded: apply platform-level adaptations (see below), then write
   the result to the destination path. Create intermediate directories as
   needed.

### Resolving platform conflicts

Port each file verbatim. When you encounter something that cannot be carried
over as-is — a component, import, API, or pattern that does not exist on the
destination platform — work through this sequence:

**Step 1 — Check approved exceptions**

Read the `## Sync` section of the destination CLAUDE.md. If there is already
an approved exception that covers this conflict (e.g. "use `<Button>` from
`@ds/mobile` wherever source uses `<Button>` from `@ds/web`"), apply it and
continue. Do not surface it to the user again.

**Step 2 — Investigate the destination codebase**

If there is no approved exception, search the destination codebase for how
the same kind of thing is handled elsewhere:
- Look in files adjacent to the destination file (same feature, same directory)
- Look in other already-synced feature pairs if they exist (find CLAUDE.md
  files with sync metadata)
- Look for shared components or utilities in the project's design system or
  shared packages

The goal is to find the established destination-platform pattern for this
kind of construct, not to invent a new one.

**Step 3 — Surface the conflict to the user**

Present what you found:

> "Conflict in `<file>`:
> - Source uses: `<what the source has, with import path>`
> - Found in destination codebase: `<what you found, with file reference>`
>
> Should I use `<found alternative>` here, or something else?"

Do not guess or proceed without confirmation. The user may correct the
proposed alternative.

**Step 4 — Verify and confirm**

If the user provides or corrects the alternative, verify that the proposed
pattern exists in the codebase and would work in context. If something seems
off (missing import, wrong API shape), raise it rather than silently using it.
Continue iterating until the user explicitly confirms the resolution.

**Step 5 — Record and continue**

Once confirmed, add the resolution to the destination CLAUDE.md under
"Approved exceptions". Then apply it to the current file and
continue porting. The same exception must be applied consistently across all
remaining files in this sync run.

### Handling new vs updated files

- **File exists in source, exists in destination**: overwrite destination with
  the ported source content (respecting exceptions/exclusions).
- **File exists in source, missing in destination**: create it.
- **File exists in destination, missing in source**: do NOT delete it
  automatically. Add it to the sync report under "Files in destination not
  found in source — review manually". The user may have added platform-specific
  files that should be preserved.

### Sync report

After completing, print a structured report:

```
## Sync report: <source> → <destination>

**Files synced:** <count>
**Files skipped (excluded):** <count>
  - <file>: <reason>

**Files created (new in destination):** <count>
  - <list>

**Files in destination not found in source — review manually:**
  - <list, or "None">

**Exceptions applied (from approved list):**
  - <source pattern> → <destination pattern>: applied to <n> files

**New conflicts resolved this run:**
  - <source pattern> → <destination pattern>: confirmed by user, added to approved exceptions
```


