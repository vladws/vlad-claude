---
name: sync
description: >
  Port a mobile feature flow to web, maintaining a verbatim 1-to-1 sync
  between a mobile source directory and a web destination directory. Use when
  the user invokes /sync, asks to "sync mobile to web", "port this feature to
  web", "propagate mobile changes to web", or says something like "keep mobile
  and web in sync". Also trigger when the user asks to set up a sync
  relationship between a mobile and web feature folder, or when they've changed
  the mobile flow and want the web version updated. Do NOT trigger for
  web-to-mobile direction, git sync, database sync, or unrelated file
  operations.
---

Port a mobile feature flow to web, maintaining a verbatim 1-to-1 sync. The
direction is always **mobile → web**. The mobile directory is the source of
truth; the web directory is the destination.

Runs in four phases: identify flows, set up documentation, assess scope, then
perform the sync.

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
>  - Source (mobile): `src/features/e-transfer/mobile/`
>  - Destination (web): `src/features/e-transfer/web/`
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

## Phase 3 — Assess scope

Before porting any files, identify which screens need to be included in this
sync run. All logic within the agreed scope is ported verbatim — this phase
only determines which screens are in scope.

### Identify screens in the source

Scan the mobile source directory for screen-level components. A screen is a
top-level component registered in a navigator or exported as an entry point
for a route. Look at:
- Navigator files (they enumerate registered screens explicitly)
- Top-level component files that are imported by navigators
- A `screens/` subdirectory if one exists

Build a flat list: screen name → source file path.

### Filter against existing exclusions

Read the destination CLAUDE.md. Remove from the list any screens already
recorded under "Explicit exclusions (user-confirmed)". These have been
previously decided and do not need to be re-asked.

### Check which screens are new

For each remaining screen, check whether a corresponding file already exists
in the destination directory (same relative path). Split the list into:
- **Already ported**: file exists in destination
- **New**: file does not exist in destination

Do not present already-ported screens to the user — they are in scope by
default (the sync will update them).

### Confirm new screens with the user

Present the new screens and ask which to include:

> "The following screens exist in mobile but have no web equivalent yet:
>
> - `ConfirmationScreen` (`screens/confirmation-screen.tsx`)
> - `ReviewScreen` (`screens/review-screen.tsx`)
> - `SummaryScreen` (`screens/summary-screen.tsx`)
>
> Which of these should I port in this run? (Answer with all, none, or a
> list)"

For any screen the user excludes, add it to "Explicit exclusions
(user-confirmed)" in the destination CLAUDE.md before proceeding. The
sub-components of an excluded screen are also excluded — do not port them.

Do not proceed to Phase 4 until the user has confirmed scope.

---

## Phase 4 — Perform the sync

Walk the source directory and produce a verbatim port in the destination.

### The data flow invariant (highest priority)

The sequence of data flow — what queries run, in what order, behind what
conditions, and when their cache gets invalidated — **is** the feature. Two
platforms that render identical pixels but fetch or invalidate data
differently are not in sync; they will diverge in behavior the moment a
backend response is slow, a user backgrounds the app, or a mutation lands.
Preserve this sequence before anything else.

Concretely, when porting any file that touches data:

- **Conditional queries.** If the source fires a query inside a conditional
  branch — `{condition && <ComponentThatQueries />}`, a hook called from a
  child that only mounts when a flag is true, a query guarded by a
  `skip`/`enabled` argument tied to upstream state — the destination must
  preserve the same conditional structure. Do not hoist the query to a
  common parent and pass data down. Do not turn a gated query into an
  always-on one. Do not flatten the condition by rendering the component
  unconditionally and "handling it later". The query must fire under the
  same conditions, in the same place, at the same point in the render tree.

- **Cache invalidation.** If the source invalidates a cache key at a
  specific point — after a particular mutation, on unmount of a particular
  screen, inside a particular conditional branch, after a specific user
  action — the destination must invalidate the same key at the same point.
  Do not relocate invalidations to a "more convenient" hook. Do not
  consolidate multiple invalidations into one. Do not omit them because the
  destination platform "would refetch anyway".

- **Order of operations.** Queries, mutations, and effects that run in a
  particular order in the source must run in the same order in the
  destination. Hooks that depend on each other's results must remain in the
  same relative position. If the source reads value A, then conditionally
  fires query B based on A, then invalidates key C after B resolves, the
  destination must do the same three things in the same order.

Before writing any destination file, trace its data flow: list every query,
the condition guarding it, and every invalidation point it triggers. Carry
that list to the destination unchanged. If any item cannot be carried over
verbatim (because the destination platform lacks the equivalent API), treat
it as a platform conflict and follow the "Resolving platform conflicts"
steps below — do not silently restructure.

### What "verbatim" means

Beyond the data flow invariant above, the following must also be preserved
exactly:
- File and folder names
- Component hierarchy and nesting
- The order of operations within a component (hooks called in the same order,
  effects running in the same sequence, renders structured the same way)
- Logic that spans multiple files must remain in the same relative positions
  across those files — do not collapse two source files into one or split one
  into two without explicit user confirmation

The following are adapted for the destination platform (not preserved
verbatim): UI primitive components, navigation APIs, platform-specific SDKs.
Everything else is ported as-is.

### Determine the sync scope

Do not walk the entire source directory. The sync covers only the feature's
source files — the transitive closure of what the source's `index.ts` imports.

1. Read `<source>/src/index.ts` (or `<source>/index.ts` if no `src/`
   subdirectory exists). This is the entry point.
2. Collect every file imported — directly or transitively — by the exports in
   that entry point. This set is the sync scope.
3. Exclude from scope regardless of imports:
   - Ambient type declaration files (`*.d.ts`)
   - The entry point file itself (`index.ts`) — its shape differs per platform
   - Test setup, config, and build files (`jest.config.*`, `tsconfig.*`,
     `project.json`, `*.settings.json`, `test-setup.*`)

Only files in this computed scope are candidates for syncing.

### File-level mapping

For every file in the sync scope:
1. Compute the corresponding destination path by replacing the source root with
   the destination root (preserving all intermediate directories).
2. If the file matches an explicit user-confirmed exclusion from the destination
   CLAUDE.md, skip it and note it in the sync report.
3. If the file's implementation is platform-specific (e.g. a React Navigation
   navigator, a native wallet integration), it is NOT skipped — it still exists
   in the destination as a structural equivalent. Surface the conflict to the
   user following the "Resolving platform conflicts" steps below, then create
   the destination-platform version that preserves the same structural role.
4. If no conflict: write the ported content to the destination path. Create
   intermediate directories as needed.

### Resolving platform conflicts

Port each file verbatim. When you encounter something that cannot be carried
over as-is — a component, import, API, or pattern that does not exist on the
destination platform — work through this sequence.

The required process is strictly sequential: **one file → one conflict → one
user response → continue**. There is no lookahead, no accumulation, no
batching.

Concretely:
1. Pick the next file to port.
2. Read it and attempt to port it verbatim.
3. If you hit a conflict **in that file**, stop immediately — do not read any
   other files to find more conflicts first.
4. Resolve that single conflict with the user (see steps below).
5. Finish porting the current file.
6. Move to the next file and repeat from step 2.

Do not "find all the platform conflicts" before starting. Do not accumulate a
list of conflicts across files and present them together. Each conflict gets
its own isolated exchange with the user — one conflict per message, one
message per conflict.

**Step 1 — Check approved exceptions**

Before surfacing a conflict, read the `## Sync` section of the destination
CLAUDE.md. If an approved exception already covers this conflict, apply it
silently and continue. Do not surface it to the user again.

**Step 2 — Investigate the destination codebase**

If no approved exception exists, search the destination codebase for the
established pattern for this kind of construct:
- Files adjacent to the destination file (same feature, same directory)
- Other already-synced feature pairs (find CLAUDE.md files with sync metadata)
- Shared components or utilities in the project's design system or shared
  packages

The goal is to find what already exists in the destination, not to invent a
new approach.

**Step 3 — Surface the single conflict**

Send one message containing exactly one conflict:

> "Conflict in `<file>`:
> - Source uses: `<what the source has, with import path>`
> - Found in destination codebase: `<what you found, with file reference>`
>
> Should I use `<found alternative>` here, or something else?"

Wait for the user's response before doing anything else — do not read ahead,
do not port the next file, do not look for more conflicts.

**Step 4 — Verify and confirm**

Check that the user's confirmed alternative exists in the codebase and fits
the context. If something seems wrong (missing import, wrong API shape), say
so. Keep iterating until the user explicitly confirms.

**Step 5 — Record and continue**

Only after explicit confirmation: add the resolution to the destination
CLAUDE.md under "Approved exceptions". Apply it to the current file. Then move
to the next file. Apply the same exception automatically to any subsequent
file that has the same conflict — do not re-ask for a conflict that is already
approved.

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


