---
name: ticket
description: >
  Create a Linear ticket from currently staged changes. Use when user invokes
  /ticket or asks to create a Linear ticket from staged git changes.
---

Create a Linear ticket from currently staged changes:

1. Run `git diff --staged` to read the full staged diff.

2. Create a Linear ticket using the `mcp__linear__save_issue` tool:
   - Team: infer from the staged files and project context
   - Title: short description of the staged changes
   - Description: summary of what changed and why, based on the staged diff
   - Assign to me (`assignee: "me"`)
   - Priority: Normal (3)

3. Print the returned ticket identifier (e.g. `MMVT-1234`).
