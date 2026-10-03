---
name: us-to-specs
description: Break a refined user story into independently implementable vertical-slice specs (tracer bullets) under `.specs/`, with an implementation-order file. Use after /refine-us and before an /implement-specs-* skill (nestjs, nextjs, react-native).
argument-hint: "[USER_STORY_FILE]"
---

USER_STORY_FILE: $ARGUMENTS[0]

# 1. Load the inputs

1. Read the user story file. If it has no `## Technical Refinement` section, say so and
   recommend running `/refine-us` first — then continue only if the developer insists.
2. Number the acceptance criteria **in order of appearance** (`AC1 … ACn`) and keep that
   numbering everywhere; it is how specs reference the story.
3. Skim the codebase for the precedent the story mirrors (a sibling feature, an earlier
   ticket in `.specs/`) so slice boundaries follow the project's existing seams.
4. If the story links a design file and a design MCP server is available, read the screens
   it references before slicing a frontend story.

# 2. Draft vertical slices

Each slice is a **tracer bullet**: a thin path through every layer the feature touches
(schema → service → endpoint/UI → tests), demoable or verifiable on its own.

Heuristics:

- Prefer **4–8 slices**. Fewer than 3 means the story is a bugfix — one spec is fine. More
  than 10 means slices are horizontal; merge until each one is demoable.
- A slice should be finishable in about a day of focused work.
- **Enabler slices** (a migration, a shared util extraction, a contract change) are allowed
  only when they unblock two or more slices or must land before anything else. Otherwise
  fold the work into the first slice that needs it.
- The first slice that touches the feature's main flow should be the **scaffold**: the
  narrowest end-to-end path (one field, one happy case) so the architecture is validated
  before the rules pile on.
- Put documentation/rule updates in the slice that changes the behaviour they describe, and
  a final doc sweep in the last slice.
- **AFK** = can be implemented and merged without a human decision. **HITL** = a human must
  decide or review first (design sign-off, contract change visible to clients, data
  migration on production data). Mark HITL only for decisions, not for "it's big".

# 3. Quiz the developer

Present the breakdown as a table:

| # | Title | Type | Blocked by | ACs covered |
|---|-------|------|------------|-------------|

Then ask, in one message:

1. Granularity — too coarse / too fine? Any slice to merge or split?
2. Dependencies — are the "blocked by" relationships right? Which slices can run in parallel?
3. HITL — is anything marked AFK that needs a human first (or vice versa)?

Also list any **defaults you assumed** from the story's open questions so the developer can
flip them. Accept a plain "ok" / "looks good" as approval. Iterate until approved.

# 4. Write the specs

1. Ensure `.specs/` exists at the project root; create it if not.
2. Create `.specs/<TICKET>-<kebab-case-title>/` (ticket id from the story file name or its
   header; fall back to the kebab title alone).
3. For each slice, in dependency order (blockers first), create
   `NN-<kebab-case-slice-title>.md` from `templates/spec-template.md` (in this skill's
   folder). Fill every section; reference ACs by number; name real files and functions where
   the codebase already has them; put the precedent the slice mirrors in **What to build**.
4. Create `implementation-order.md` from `templates/implementation-order-template.md`.
   Leave **Completed specs** as `_(none yet)_`. In **Notes**, record contract changes,
   migration ordering, "never do X" rules from the story, and the commit/branch convention
   the project uses (read it from `CLAUDE.md` or the git log).

Timestamps use `YYYY-MM-DD`. The first log line of a spec is `Ready to start` when it has
no blockers and `Blocked (NN, NN)` otherwise.

# 5. Hand off

Print the folder path, the table, and the next command, picking the implementation skill
that matches the project's stack (detect it from `package.json`: `@nestjs/core` → nestjs,
`next` → nextjs, `react-native` / `expo` → react-native):

`/implement-specs-<stack> .specs/<folder>/implementation-order.md`

# Rules

- **Never modify the parent user story document.**
- Do not write code in this skill. Specs describe behaviour and name the touch points; the
  implementation skill decides the details.
- Keep each spec self-contained: a developer who has not read the story must be able to
  implement it from the spec plus the referenced sections.
