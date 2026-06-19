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

Port a mobile **modal flow** to web as a verbatim 1-to-1 sync. The web
destination is a **multi-step modal flow** with the same shape. Direction is
always **mobile → web**: mobile is the source of truth, web is the destination.

The skill branches on whether a sync relationship already exists:

- **First run** (no `## Sync` in destination CLAUDE.md): establish the
  contract, agree on scope, port.
- **Re-run** (sync section exists): detect scope drift, then content drift,
  reconcile both.

Shared mechanics (orchestrator vs screens, verbatim porting, data flow
invariant, platform conflicts, sync report) are defined once in "Sync
mechanics" at the bottom and referenced by both paths.

---

## Domain model

Both source and destination are **modal flows**: a wrapping modal that
progresses the user through a sequence of pages. The two sides share the
same shape, with different names for the unit of progression:

- **Orchestrator layer** — modal shell, providers, navigator (mobile) or
  stepper (web), shared hooks, utilities, state, types. Holds the flow
  together; routes nothing on its own without screens to plug into. Always
  synced verbatim because partial orchestrator sync produces a broken flow.
- **Screens** (mobile) ↔ **modal steps** (web) — the discrete pages the user
  moves through. Opt-in for sync; each is a unit the user can include or
  exclude.
- **Sub-flows** — a nested modal flow rendered from inside a screen. On
  mobile they appear either inline within the parent modal or as a separate
  modal stack. On web they mount as steps inside the parent stepper. A
  sub-flow's screens form a tree rooted under the parent screen: when the
  parent screen is in scope, the sub-flow's screens come with it (treat
  them as part of the parent's sub-tree), unless explicitly excluded.

Throughout this skill, **"screen" means a unit of the flow on either side**
— mobile "screen" and web "modal step" name the same concept. Use whichever
term the surrounding code uses; the sync treats them identically.

---

## Phase 1 — Identify source and destination

**Explicit paths from the user:** confirm before proceeding.

> "Syncing from `<source>` → `<destination>`. Correct?"

**Vague request (e.g. "sync the e-transfer flow"):** search common locations
(`src/features/`, `src/flows/`, `src/screens/`, `src/pages/`, `apps/`,
`packages/`) plus any directory containing a `CLAUDE.md` with sync metadata.
Present candidates with full paths and wait for confirmation.

A wrong sync target can corrupt a platform's codebase — never assume.

---

## Phase 2 — Detect run mode

Read `<destination>/CLAUDE.md`. If a `## Sync` section exists, this is a
**re-run** (go to Phase 3B). Otherwise, **first run** (Phase 3A). Derive
mechanically; don't ask the user.

If the section exists but is missing required fields (in-scope screens,
orchestrator description), treat as first run and tell the user:

> "Found `## Sync` in `<destination>/CLAUDE.md` but required fields are
> missing. Treating this as a first run and rebuilding the contract."

---

## Phase 3A — First run: establish contract and scope

### Step 1 — Compute the orchestrator layer

The **orchestrator layer** is defined in "Domain model" above. To compute it
for this feature:

1. Read `<source>/src/index.ts` (or `<source>/index.ts`). This is the entry
   point.
2. Collect every transitively imported file — the full sync scope (see "Sync
   scope computation" below).
3. Within that set, identify **screen files**: top-level components
   registered in the modal flow's navigator/stepper, exported as route
   entry points, or living under `screens/`. Components belonging to a
   sub-flow nested inside a screen are part of that screen's sub-tree —
   they are not separate top-level screens.
4. Orchestrator = sync scope − screen files − each screen's sub-tree
   (which includes any sub-flow's screens rendered from within it).

### Step 2 — List candidate screens

```
- ConfirmationScreen     screens/confirmation-screen.tsx
- ReviewScreen           screens/review-screen.tsx
- SummaryScreen          screens/summary-screen.tsx
```

### Step 3 — Confirm scope with the user

> "Setting up the sync contract for the `<feature>` modal flow:
>
> **Orchestrator (always synced):** modal shell, providers, navigator/
> stepper, shared hooks, state, types — `src/index.ts`, `src/providers/`,
> `src/hooks/`, `src/state/`, `src/types/`, `src/utils/`, navigator files
>
> **Screens / modal steps (opt-in):**
>  - `ConfirmationScreen` (`screens/confirmation-screen.tsx`)
>  - `ReviewScreen` (`screens/review-screen.tsx`)
>  - `SummaryScreen` (`screens/summary-screen.tsx`) — contains the
>    `IdentityVerification` sub-flow (its screens mount as steps under
>    this one)
>
> Which screens should I port now? (all / none / list) Unselected ones get
> recorded as exclusions."

Sub-components and sub-flow screens of an excluded screen are also excluded
— they move with their parent. Wait for confirmation.

### Step 4 — Write CLAUDE.md files

Use the templates under "CLAUDE.md templates". Fill in orchestrator
description, in-scope screens, and exclusions.

### Step 5 — Sync

Walk orchestrator + in-scope screens through "Sync mechanics", then emit the
sync report.

---

## Phase 3B — Re-run: drift detection and reconciliation

### Step 1 — Migrate CLAUDE.md format if needed

The destination `CLAUDE.md` `## Sync` section may have been written by an
older version of this skill, with a different structure (for example: flat
"Explicit exclusions" and "Feature-specific overrides" lists at the top
level, no `### Exceptions` heading, no per-screen `#### Screen: <Name>`
groups, no "Screens in source excluded from sync" / "Screens in destination
not present in source" sections). Before reading scope or exceptions,
compare the existing structure to the template in "CLAUDE.md templates"
below. The current template requires:

- `### Orchestrator layer (always synced verbatim)` heading
- `### In-scope screens` heading
- `### Exceptions` heading with these subsections:
  - `#### Orchestrator layer`
  - One `#### Screen: <Name>` per in-scope screen
  - `#### Screens in source excluded from sync`
  - `#### Screens in destination not present in source`

If any required heading is missing, the format is outdated. Migrate it
following these rules:

1. **Preserve every entry.** No exception, override, exclusion, or screen
   reference may be dropped during migration. Re-shuffle into the new
   structure — never delete.
2. **Re-slot by where each entry applies:**
   - Old flat "Explicit exclusions" entries naming screens → "Screens in
     source excluded from sync".
   - Old flat "Platform-specific files owned by this directory" entries
     naming whole destination-only screens → "Screens in destination not
     present in source". File-level entries (not whole screens) → the
     "Platform-specific files owned by destination" entry of the owning
     group (orchestrator if the file sits outside any screen tree,
     otherwise the matching `#### Screen:` group).
   - Old flat "Feature-specific overrides" → decide owning group from the
     file reference each entry cites. Move accordingly. Overrides applying
     across multiple screens go once under the orchestrator group rather
     than duplicated per screen. If ownership is ambiguous (no file
     reference, or the entry applies broadly), keep it under the
     orchestrator group and flag it in the migration summary.
3. **Add any missing required headings** using the current template,
   including the explanatory text under `### Exceptions`.
4. **Apply the same check to the source `CLAUDE.md`.** Its `## Sync` section
   is shorter — confirm the "Rules for agents working here" block matches
   the current template and the cross-reference to the destination path
   is accurate.

After migration, surface a brief summary before proceeding:

> "Migrated `<destination>/CLAUDE.md` from an older sync format. No entries
> dropped. Moved <n> exclusions, <n> overrides, <n> platform-specific
> entries. Ambiguous entries kept under orchestrator group: <list>. Review
> and refine if needed."

Omit the ambiguous-entries line if none surfaced. If the format already
matches the current template, skip this step silently.

Only after migration (or a clean skip) proceed to Step 2.

### Step 2 — Detect scope drift

From destination CLAUDE.md `## Sync`, extract:
- Recorded orchestrator description
- **In-scope screens** (each `#### Screen: <Name>` subsection counts)
- **Screens in source excluded from sync**
- **Screens in destination not present in source**

Recompute screens in source (Phase 3A Step 2 rule) and list current
destination screen files. Classify each:

| Bucket | Definition | Action |
|---|---|---|
| **In sync** | In source + destination, recorded as in-scope | Content-drift check (Step 4) |
| **New in source** | In source, not in destination, not in "Screens in source excluded from sync" | Ask user (Step 3) |
| **Orphan in destination** | In destination, not in source, not in "Screens in destination not present in source" | Ask user (Step 3) |
| **Already excluded** | Listed in either exclusion section | Leave alone |

If every screen is "In sync" or "Already excluded", skip Step 3.

### Step 3 — Reconcile scope drift

Present all drift items in one message so the user sees the full picture,
then record decisions individually.

> "Scope drift detected:
>
> **New in source:** `WelcomeScreen` (`screens/welcome-screen.tsx`)
> **Orphan in destination:** `WebOnlyHelpScreen` (`screens/web-only-help-screen.tsx`)
>
> For each:
>  - `WelcomeScreen`: port now / exclude from sync / skip this run
>  - `WebOnlyHelpScreen`: delete / preserve as destination-only / skip this run"

Persist each resolution to CLAUDE.md immediately:

- **Port now** → add to "In-scope screens" + create empty `#### Screen: <Name>`
  subsection under "Exceptions".
- **Exclude from sync** → add to "Screens in source excluded from sync" with
  reason (or `(no reason given)`).
- **Preserve as destination-only** → add to "Screens in destination not
  present in source" with reason.
- **Skip this run** → no CLAUDE.md change; drift surfaces again next run.
- **Delete** (orphan only) → confirm explicitly first, since this is
  destructive:
  > "Delete `<file>` from destination? Removes the file and any sub-components
  > only it imports. Confirm?"

Wait until every drift item has a recorded decision before moving on.

### Step 4 — Detect and resolve content drift

Walk orchestrator + all currently in-scope screens (including anything added
in Step 3). For each file:
1. Compute what the destination *should* contain by porting source through
   "Sync mechanics" with recorded mappings and overrides applied silently.
2. Compare against current destination content.
3. If they match: in sync. If they differ: overwrite destination with the
   ported source.

For divergences not covered by any recorded rule, fall back to the
platform-conflict process in "Sync mechanics" (one conflict, one file, one
user response, continue).

The destination is never a source of truth. Any unrecorded drift is treated
as accidental and overwritten. To keep destination-only behavior, the user
must record it under "Feature-specific overrides" or under one of the
"Platform-specific files" lists.

### Step 5 — Emit sync report

Standard format (see "Sync mechanics") with a drift-resolution header:

```
## Scope drift resolved this run
- <screen>: <decision> (recorded as <where>)
```

---

## Sync mechanics

### The data flow invariant (highest priority)

The sequence of data flow — what queries run, in what order, under what
conditions, and when their cache gets invalidated — **is** the feature. Two
platforms that render identical pixels but fetch or invalidate differently
will diverge the moment a backend response is slow, a user backgrounds the
app, or a mutation lands. Preserve this sequence before anything else.

When porting any file that touches data:

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

Before writing any destination file, list every query, the condition guarding
it, and every invalidation it triggers. Carry that list across unchanged. If
something cannot be carried verbatim, treat it as a platform conflict — do
not silently restructure.

### What "verbatim" means

Beyond the data flow invariant, preserve exactly:
- File and folder names
- Component hierarchy and nesting
- Hook call order, effect sequencing, render structure
- Cross-file logic boundaries — don't collapse two files into one or split
  one into two without explicit user confirmation

Adapted (not verbatim): UI primitive components, navigation APIs,
platform-specific SDKs. Everything else ports as-is.

### Sync scope computation

The sync covers only the feature's transitively imported files — not the
whole source directory.

1. Read `<source>/src/index.ts` (or `<source>/index.ts`). This is the entry
   point.
2. Collect every file transitively imported by its exports.
3. Exclude regardless of imports:
   - Ambient declarations (`*.d.ts`)
   - The entry point itself (shape differs per platform)
   - Test setup, config, and build files (`jest.config.*`, `tsconfig.*`,
     `project.json`, `*.settings.json`, `test-setup.*`)

Orchestrator + in-scope screens (and their sub-trees) is a subset of this.

### File-level mapping

For each file in the active scope (orchestrator + in-scope screens):
1. Map source path → destination path by swapping roots, preserving
   intermediate directories.
2. If the file matches a recorded exclusion, skip and note in the report.
3. If the implementation is platform-specific (e.g. a React Navigation
   navigator, a native wallet integration), don't skip — surface as a
   platform conflict, then create a destination-platform version that fills
   the same structural role.
4. Otherwise: write the ported content, creating directories as needed.

### Resolving platform conflicts

Port verbatim. When you hit something that cannot be carried over as-is — a
component, import, API, or pattern absent on the destination platform — work
through the steps below.

**The process is strictly sequential: one file → one conflict → one user
response → continue.** No lookahead, no accumulation, no batching. Each
conflict is its own message exchange.

1. Pick the next file.
2. Attempt to port verbatim.
3. On the first conflict in that file, stop. Do not scan ahead.
4. Resolve with the user (steps below).
5. Finish the current file.
6. Next file.

**Step 1 — Check generic mobile-to-web mappings**

Consult the `mobile-to-web-mappings` skill in the `design-system` plugin.
It owns generic mobile→web translations that apply across every synced
feature: design-system primitives (mobile `<Button>` → web `<Button>`,
`<Touchable>` → `<Pressable>`), navigation APIs, platform shims, image
handling. If it matches, apply silently — do not record in the feature
CLAUDE.md (duplicating generic mappings creates drift the moment the design
system changes).

If the skill is unavailable, proceed as if no mapping was found and note it
in the sync report.

**Step 2 — Check feature-specific overrides**

Identify which exception group owns the file: orchestrator layer, or the
specific screen whose tree contains it. Check that group's
"Feature-specific overrides" first, then fall back to the orchestrator
group (screens inherit orchestrator-level overrides). If matched, apply
silently.

**Step 3 — Investigate the destination codebase**

If nothing matches, search for the established pattern in:
- Files adjacent to the destination file
- Other synced feature pairs (`CLAUDE.md` files with sync metadata)
- Shared design-system / utility packages

Find what already exists; don't invent.

**Step 4 — Surface the single conflict**

One message, one conflict:

> "Conflict in `<file>`:
> - Source uses: `<source pattern with import path>`
> - Found in destination: `<alternative with file reference>`
>
> Use `<alternative>`, or something else?"

Wait. Do not read ahead, port the next file, or look for further conflicts.

**Step 5 — Verify and confirm**

Check the user's choice exists and fits (imports resolve, API shape matches).
Push back if something looks wrong. Iterate until explicit confirmation.

**Step 6 — Classify and record**

After confirmation:

- **Generic mapping** (applies across features — design-system swap,
  navigation substitution, platform shim): don't add to feature CLAUDE.md.
  Tell the user:
  > "Looks generic, not feature-specific. Recommend adding to
  > `mobile-to-web-mappings`. Flag it in the sync report?"

  Apply to the current file plus any other file in this run with the same
  conflict; list under "Generic mappings to promote" in the report.

- **Feature-specific override** (only meaningful for this feature — copy
  difference, business-rule deviation, one-off integration): record under
  the **owning group's** "Feature-specific overrides" — orchestrator group
  if the file is outside any screen, otherwise the `#### Screen: <Name>`
  group. If the same override would apply across multiple screens, record
  it once at the orchestrator level instead of duplicating. Include a brief
  reason. Apply to subsequent matching files this run; don't re-ask.

When unsure, ask the user one short classification question.

### Handling new vs updated files

- **In source + in destination:** overwrite with ported source (respecting
  exclusions and overrides).
- **In source, missing in destination:** create.
- **In destination, missing in source:** never delete automatically. First
  run: list under "Files in destination not found in source — review
  manually". Re-run: handled by Phase 3B Step 2 (scope drift detection).

### Sync report

```
## Sync report: <source> → <destination>

**Run mode:** <first run | re-run>

**CLAUDE.md format migration:** (re-run only, omit if format was current)
  - Migrated `<destination>/CLAUDE.md` to the latest sync format. Moved
    <n> exclusions, <n> overrides, <n> platform-specific entries. No
    entries dropped. Ambiguous entries kept under orchestrator group:
    <list, or "None">.

**Scope drift resolved this run:** (re-run only)
  - <screen>: <decision> (recorded as <where>)

**Files synced:** <count>
**Files skipped (excluded):** <count>
  - <file>: <reason>

**Files created (new in destination):** <count>
  - <list>

**Files in destination not found in source — review manually:**
  - <list, or "None">

**Generic mappings applied (from `mobile-to-web-mappings` skill):**
  - <source pattern> → <destination pattern>: applied to <n> files
  - (note "skill unavailable in this environment" if it couldn't be consulted)

**Feature-specific overrides applied (from destination CLAUDE.md):**
  - <source pattern> → <destination pattern>: applied to <n> files

**New feature-specific overrides recorded this run:**
  - <source pattern> → <destination pattern>: added to feature CLAUDE.md
    (reason: <why feature-specific>)

**Generic mappings to promote (suggest adding to `mobile-to-web-mappings`):**
  - <source pattern> → <destination pattern>: confirmed this run, not yet
    in the design-system skill
```

---

## CLAUDE.md templates

These templates are the long-lived contract between source and destination.
Phase 3A writes them fresh. Phase 3B reads them, updates the in-scope screens
and exclusion lists as drift is resolved, and otherwise preserves them.

If a CLAUDE.md already has unrelated content, preserve it and only
append/replace the `## Sync` section.

### Source directory template

```markdown
## Sync

This directory is the **source of truth** for the `<feature name>` flow.

**Synced to:** `<relative path to destination>`

For approved exceptions, exclusions, and the in-scope screens list, see the
destination CLAUDE.md at `<relative path to destination>/CLAUDE.md`.

### Rules for agents working here
- After modifying any logic (components, hooks, utilities, types) in this
  directory, you MUST run `/sync` to propagate changes to the destination.
  If the `sync` skill is not available, notify the user and ask them to check
  `CODEOWNERS` for this directory to contact the responsible team.
- Do not let logic diverge between source and destination — the sync must be
  maintained verbatim. Exceptions are listed in the destination CLAUDE.md.
```

### Destination directory template

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

Generic mobile→web translations (design-system primitives, navigation APIs,
platform shims, and any mapping that applies across every synced feature)
are NOT recorded here. They live in the `mobile-to-web-mappings` skill in
the `design-system` plugin and are applied automatically by `/sync`. Only
record exceptions in the section below if they are **specific to this
feature** — a deviation that does not apply to other synced features.

### Orchestrator layer (always synced verbatim)
<describe the orchestrator file set in a few lines — modal shell, providers,
navigator (mobile) / stepper (web), shared hooks, state, types. E.g.: "entry
point, modal shell at src/modal/, providers under src/providers/, shared
hooks under src/hooks/, navigator at src/navigation/, types under src/types/".
This is the set of files /sync re-syncs unconditionally on every run; the
user does not opt into it screen-by-screen.>

### In-scope screens (modal steps on web)
<list each screen currently part of the sync. Sub-flow screens are part of
their parent screen's sub-tree and are not listed separately unless the user
has explicitly excluded sibling screens within the same sub-flow. E.g.:>
- `ConfirmationScreen` (`screens/confirmation-screen.tsx`)
- `ReviewScreen` (`screens/review-screen.tsx`)
- `SummaryScreen` (`screens/summary-screen.tsx`) — contains the
  `IdentityVerification` sub-flow

### Exceptions

Exceptions are grouped by **where they apply**. When porting a file, `/sync`
locates the group that owns it (orchestrator layer or a specific screen) and
applies entries from that group only. A file is owned by a screen if it lives
under the screen's sub-tree or is imported exclusively by that screen;
everything else (entry point, providers, navigators, shared hooks, utilities,
state, types, and any component reachable from more than one screen) belongs
to the orchestrator layer.

#### Orchestrator layer

**Feature-specific overrides (confirmed by user during sync runs):**
<list each override as: "source uses X → destination uses Y (file reference where pattern was found, brief reason it's specific to this feature)"; or "None" if none yet>

**Platform-specific files owned by destination (not synced):**
<list orchestrator-layer files that exist only on the destination platform — e.g. platform navigation wrappers, wallet integrations; or "None">

#### Screen: `<ScreenName>` (`<path>`)

**Feature-specific overrides:**
<list, or "None">

**Platform-specific files owned by destination (not synced):**
<list sub-components or helpers under this screen's tree that exist only on the destination platform, or "None">

<repeat one section per in-scope screen>

#### Screens in source excluded from sync

<list each excluded source screen as: "`<ScreenName>` (`<source path>`) — <reason>"; or "None">

#### Screens in destination not present in source

<list each destination-only screen as: "`<ScreenName>` (`<destination path>`) — <reason it is preserved>"; or "None">
```
