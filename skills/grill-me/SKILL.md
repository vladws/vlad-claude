---
name: grill-me
description: >
  Conduct a relentless Socratic interview about a plan or design, walking down
  each branch of the decision tree until reaching shared understanding, then
  capture the outcomes as a PRD. Use only when the user explicitly invokes
  /grill-me.
---

Interview me relentlessly about every aspect of this plan until we reach a shared understanding. Walk down each branch of the decision tree, resolving dependencies between decisions one-by-one. For each question, provide your recommended answer.

Ask the questions one at a time.

If a question can be answered by exploring the codebase, explore the codebase instead of asking.

## Wrapping up

When you feel all major branches of the decision tree have been covered, propose ending the session: "I think we've covered the key decisions — ready to write up the PRD?" The user can continue grilling if they want more depth.

## Writing the PRD

Once the session ends:

1. **Confirm the output directory.** Default is `.ai/plans/` in the current project root. Ask if the user wants a different location.

2. **Determine the filename.** Use a date-prefixed slug derived from the topic: `YYYY-MM-DD-feature-name.md`. Confirm with the user if the topic is ambiguous.

3. **Write the PRD** using the template below. It should capture the distilled decisions and understanding from the session — not a transcript.

### PRD template

```markdown
# [Feature / Initiative Name]

## Problem Statement

What problem is being solved, and why it matters.

## Proposed Solution

A clear description of the solution, from a product and technical perspective.

## Implementation Decisions

Key decisions reached during this session. Include:

- Modules to build or modify, and their interfaces
- Architectural decisions and trade-offs
- Schema or data model changes
- API contracts or integration points
- Specific behaviors and edge cases resolved

Do NOT include specific file paths or code snippets — they go stale quickly.

## Testing Decisions

- What makes a good test for this area (test external behavior, not implementation details)
- Which modules or behaviors will have tests
- Any prior art in the codebase to reference

## Out of Scope

What is explicitly not covered by this plan.

## Open Questions

Anything unresolved from this session that needs follow-up.
```
