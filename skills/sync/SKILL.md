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

Do not enumerate the orchestrator scope in the message — the orchestrator
is always synced and listing it invites stale documentation. Ask only about
screens:

> "Setting up the sync contract for the `<feature>` modal flow. The
> orchestrator (modal shell, providers, navigator/stepper, shared hooks,
> state, types) always ports as-is.
>
> Screens / modal steps (opt-in):
>  - `ConfirmationScreen` (`screens/confirmation-screen.tsx`)
>  - `ReviewScreen` (`screens/review-screen.tsx`)
>  - `SummaryScreen` (`screens/summary-screen.tsx`) — contains the
>    `IdentityVerification` sub-flow (its screens mount as steps under
>    this one)
>
> Which screens port now? (all / none / list) Unselected screens get a
> per-screen section marked `— out of scope`."

Sub-components and sub-flow screens of an excluded screen move with their
parent. Wait for confirmation.

### Step 4 — Write CLAUDE.md files

Use the templates under "CLAUDE.md templates". Write each screen as its own
`#### Screen: <Name> — in scope|out of scope` section. Do not produce a
separate "in-scope screens" list — the per-screen headings carry status.

### Step 5 — Sync

Walk orchestrator + in-scope screens through "Sync mechanics", then emit the
sync report.

---

## Phase 3B — Re-run: drift detection and reconciliation

### Step 1 — Migrate CLAUDE.md format if needed

The destination `CLAUDE.md` `## Sync` section may have been written by an
older version of this skill. Before reading scope or exceptions, compare
its structure to the template in "CLAUDE.md templates" below. The current
template requires:

- `### Source of truth: mobile only` heading
- `### Exceptions` heading with these subsections:
  - `#### Orchestrator layer`
  - One `#### Screen: <Name> — in scope|out of scope|destination-only` per
    screen (status carried inline on the heading)

If any required heading is missing, the format is outdated. Migrate it
following these rules:

1. **Preserve every entry.** No exception, override, exclusion, or screen
   reference may be dropped during migration. Re-shuffle into the new
   structure — never delete.
2. **Re-slot by where each entry applies:**
   - Old `### In-scope screens` list → drop the list. Each named screen
     becomes a `#### Screen: <Name> — in scope` heading.
   - Old `### Orchestrator layer (always synced verbatim)` descriptive
     paragraph → drop. The orchestrator scope is implicit.
   - Old `#### Screens in source excluded from sync` entries → each becomes
     a `#### Screen: <Name> — out of scope` heading with the recorded
     reason as its body.
   - Old `#### Screens in destination not present in source` entries →
     each becomes a `#### Screen: <Name> — destination-only` heading.
   - Old flat "Explicit exclusions" or "Feature-specific overrides" entries
     → decide owning group from the file reference each entry cites. Move
     to that group. Overrides applying across multiple screens go once
     under the orchestrator group rather than duplicated per screen.
     Ambiguous entries (no file reference) stay under the orchestrator
     group and surface in the migration summary.
   - Old "Platform-specific files owned by destination" file-level entries
     → fold into the owning group's bullet list.
3. **Add the `### Source of truth: mobile only` heading** if missing,
   using the template wording.
4. **Apply the same check to the source `CLAUDE.md`.** Its `## Sync`
   section is shorter — confirm the "Rules for agents working here" block
   matches the current template and the cross-reference to the destination
   path is accurate.

After migration, surface a brief summary:

> "Migrated `<destination>/CLAUDE.md` from an older sync format. No entries
> dropped. Moved <n> exclusions, <n> overrides, <n> platform-specific
> entries. Ambiguous entries kept under orchestrator group: <list>. Review
> and refine if needed."

Omit the ambiguous-entries line if none surfaced. If the format already
matches the current template, skip this step silently.

Only after migration (or a clean skip) proceed to Step 2.

### Step 2 — Detect scope drift

From destination CLAUDE.md `## Sync`, walk every `#### Screen: <Name> —
<status>` heading. The status suffix (`in scope`, `out of scope`,
`destination-only`) is the source of truth for each screen's classification.

Recompute screens in source (Phase 3A Step 2 rule) and list current
destination screen files. Classify each:

| Bucket | Definition | Action |
|---|---|---|
| **In sync** | In source + destination, heading marked `— in scope` | Content-drift check (Step 4) |
| **New in source** | In source, no `#### Screen` heading for it | Ask user (Step 3) |
| **Orphan in destination** | In destination, no `#### Screen` heading for it | Ask user (Step 3) |
| **Already recorded** | Heading marked `— out of scope` or `— destination-only` | Leave alone |

If every screen is "In sync" or "Already recorded", skip Step 3.

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

- **Port now** → create a `#### Screen: <Name> — in scope` section under
  "Exceptions" (empty body if no overrides yet).
- **Exclude from sync** → create `#### Screen: <Name> — out of scope` with
  the reason as its body (or `(no reason given)`).
- **Preserve as destination-only** → create `#### Screen: <Name> —
  destination-only` with the reason as its body.
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

The destination is never a source of truth. Any unrecorded drift is
treated as accidental and overwritten. To keep destination-only behavior,
the user must record it as a bullet under the owning group in "Exceptions"
or as a "Platform-specific files owned by destination" entry under the
orchestrator group.

### Step 5 — Emit sync report

Use the standard format (see "Sync mechanics"). Populate the "Scope drift
resolved" and "Stale exceptions removed" sections from this phase.

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

### Grounding every exception in code

Before writing any exception entry — in the orchestrator group, a per-screen
group, or as part of platform-conflict resolution — the entry must reference
real code on **both sides**:

- The **source pattern** it replaces (file path + symbol that exists in the
  current source tree).
- The **destination alternative** it picks (file path + symbol that exists
  in the destination tree, or a clear "destination-only file" marker).

If either citation cannot be produced, the exception does not get written.
Exceptions without code grounding rot the moment the feature evolves and
make the document harder to trust on every later run. During re-runs, any
recorded exception whose cited source or destination no longer exists is
flagged for the user — "cited code is gone; keep or remove?" — never
preserved silently.

Anti-patterns to avoid:

- **Inventing exceptions.** Do not record a deviation unless porting the
  source actually surfaced one. If the source has no per-row timing logic,
  the destination doc must not claim one was dropped.
- **Carrying stale entries.** During Phase 3B, re-verify each existing
  entry against current code before re-emitting it. If the pattern it
  describes no longer appears in source, it goes.
- **Restating defaults.** Do not record entries that repeat what the sync
  contract already guarantees or what generic mappings already cover. See
  "What is not an exception" below.
- **Compound entries.** One bullet, one pattern. If you find yourself
  joining two unrelated facts with "and" or "also" inside a single bullet
  ("API replaced with X **and** haptics dropped"), split them. Compound
  entries hide overlap on re-runs and resist deduplication.

### What is not an exception

Before writing any bullet, run it past this filter. If it matches any of
these, drop it — the behavior is already covered.

- **Restating the verbatim contract.** "Locale key copied as-is",
  "component name kept the same", "logic ported unchanged" — the sync
  contract already says everything ports verbatim. Recording it as an
  exception inverts the meaning.
- **Generic mobile→web mappings.** Anything `mobile-to-web-mappings`
  already covers (design-system primitives, navigation APIs, haptics,
  animations, platform shims). These get applied silently and never
  recorded in feature CLAUDE.md.
- **Side effects of an existing entry.** If the orchestrator group already
  records "provider X mounted at entry", do not re-mention provider X in
  every per-screen entry that consumes it. Cite the provider's existence
  by reference, not by repetition.
- **Restated source-of-truth principle.** "When mobile lacks a value, use
  no-op" is covered by "Source of truth: mobile only". Per-instance
  applications of this principle ("the X variant defaults to off because
  mobile has no equivalent") get recorded only when the no-op choice is
  non-obvious enough that a future reader would re-derive it incorrectly.

### Where each exception belongs

Pick the group with the narrowest scope that still covers every use site.

- **Orchestrator group** when the deviation is infrastructure shared by
  every screen: a provider mounted at the entry, a context wrapper, a
  shared hook, a navigator rewrite, a redirect rule. Test: "if I removed
  any single screen tomorrow, would this still apply?" If yes →
  orchestrator group.
- **Per-screen group** when the deviation is specific to that screen's
  logic and would disappear if the screen were dropped: a screen-local
  component swap, a screen-local data-fetch rewrite, a screen-local copy
  difference.

When porting flags an infrastructure deviation while resolving a
screen-level file, record the infrastructure piece at the orchestrator
level and the screen-local consequence (if any) at the screen level.
Never record the same pattern in both groups.

### Before writing, dedupe

When about to record a new entry:

1. Scan the orchestrator group's bullets for the same source pattern,
   destination symbol, or theme.
2. Scan the owning screen's bullets too.
3. If a match exists, merge into the existing bullet (clarify wording,
   add a missing citation) instead of adding a new one.
4. If the new entry restates a default per "What is not an exception",
   drop it entirely.

### Feature flags

Feature-flag **names** may differ between mobile and web. When porting a
file that reads a flag whose name has no obvious web equivalent, ask the
user once for the web name, write the literal into the destination code,
and move on. Do not record specific flag mappings in CLAUDE.md — flag names
change, and a stale list outside the code becomes a trap. The orchestrator
group may carry one short rule noting that flag names are confirmed
per-sync and held in code; nothing more.

### Generic phrasing for orchestrator rules

Orchestrator-level entries describe patterns that apply across the whole
flow. Do not name specific screens or files as illustrative examples — the
screens cited go stale before the pattern does. Phrase the rule in terms of
the structure it governs ("redirect navigation targeting an out-of-scope
screen to the placeholder"), not in terms of the screens that happen to
trigger it today.

Per-screen entries are the right place for screen-specific filenames and
symbols, because they live next to the screen they describe and rot
together with it.

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

**Step 2 — Check feature-specific exceptions**

Identify which group owns the file: orchestrator layer, or the specific
screen whose tree contains it. Check that group's bullets in "Exceptions"
first, then fall back to the orchestrator group (screens inherit
orchestrator-level rules). If matched, apply silently.

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

- **Feature-specific exception** (only meaningful for this feature — copy
  difference, business-rule deviation, one-off integration): record as a
  bullet under the **owning group** in "Exceptions" — orchestrator group
  if the file is outside any screen, otherwise the `#### Screen: <Name> —
  in scope` group. If the same exception would apply across multiple
  screens, record it once at the orchestrator level (phrased generically,
  no screen-name examples) instead of duplicating. Cite the source pattern,
  the destination alternative, and a brief reason. Apply to subsequent
  matching files this run; don't re-ask.

When unsure, ask the user one short classification question.

### Handling new vs updated files

- **In source + in destination:** overwrite with ported source (respecting
  exclusions and overrides).
- **In source, missing in destination:** create.
- **In destination, missing in source:** never delete automatically. First
  run: list under "Files in destination not found in source — review
  manually". Re-run: handled by Phase 3B Step 2 (scope drift detection).

### Sync report

Keep the report short. Omit empty sections.

```
## Sync report: <source> → <destination>

**Run mode:** <first run | re-run>

**Files synced:** <count> (created: <n>, overwritten: <n>, skipped: <n>)

**Scope drift resolved:** (re-run only)
  - <screen>: <decision>

**CLAUDE.md format migrated:** (re-run only)
  - <one-line summary; ambiguous entries listed if any>

**Stale exceptions removed:** (re-run only)
  - <entry>: cited source/destination no longer present

**New feature-specific overrides recorded:**
  - <source pattern> → <destination pattern>: <reason>

**Generic mappings to promote (suggest adding to `mobile-to-web-mappings`):**
  - <source pattern> → <destination pattern>

**Files in destination not found in source — review manually:**
  - <list>
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

Do not edit files here for logic changes. Workflow:

1. Modify the source at `<source path>`
2. Run `/sync` to propagate
3. Only platform-specific adjustments listed under "Exceptions" go here
   directly

If `/sync` is unavailable, do not hand-port. Notify the user and check
`CODEOWNERS` on the source.

### Sync contract

This directory is a verbatim 1-to-1 port. Component names, file and folder
names, hierarchy, logic, utilities, variable names, and code comments match
the source exactly.

Generic mobile→web translations (design-system primitives, navigation APIs,
platform shims) live in the `mobile-to-web-mappings` skill and are applied
automatically by `/sync`. Record only **feature-specific** exceptions
below.

### Source of truth: mobile only

Mobile is the sole source of behavior for this flow. Do not borrow
decisions, default values, or business rules from other packages. When web
requires a value mobile lacks, choose the no-op or off state and surface
the gap during sync; do not invent a value.

### Exceptions

Exceptions are grouped by **where they apply**. `/sync` resolves each file
to either the orchestrator group or a specific screen. A file belongs to a
screen if it lives under that screen's sub-tree or is imported only by
that screen; everything else belongs to the orchestrator layer.

Every entry below cites a real pattern in the source and a real
alternative in the destination. Entries without code grounding rot; if the
cited code disappears, the entry goes with it.

#### Orchestrator layer

<list each orchestrator-level deviation as a bullet: name the pattern,
cite the source file/symbol, cite the destination file/symbol, give a
brief reason. Phrase rules generically — do not use specific screen names
as illustrative examples; per-screen detail belongs in per-screen
sections. One short rule on feature-flag handling lives here ("flag names
confirmed per-sync, held in code"). "None" if no deviations yet.>

**Platform-specific files owned by destination (not synced):**
<list orchestrator-layer files with no mobile counterpart — e.g. the web
entry component, platform navigation wrappers. Each entry cites the
destination file. "None" if empty.>

#### Screen: `<ScreenName>` (`<path>`) — in scope

<list per-screen deviations as bullets. Each cites a source pattern and a
destination alternative. "No screen-specific overrides." if empty.>

<repeat one `#### Screen` heading per screen the sync touches. Use the
status suffix that fits:>
- `— in scope` — ported on every run, may carry per-screen overrides
- `— out of scope` — body explains why it is not ported; no overrides
- `— destination-only` — body explains why it exists only on the
  destination side (e.g. a placeholder for redirected navigation); no
  source counterpart

<sub-flow screens belong inside their parent screen's section unless the
user has explicitly carved them out.>
```
