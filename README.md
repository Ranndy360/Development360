# development360

A [Claude Code](https://docs.claude.com/en/docs/claude-code) plugin for **specs-driven
development**: take a user story from a vague idea to implemented, tested code in three
explicit stages, with a human decision point between each.

```
story.md ──▶ /refine-us ──▶ /us-to-specs ──▶ /implement-specs-nestjs ──▶ commits
             (decide)        (slice)           (build · test · lint)
```

## What's inside

| Path | What it is |
|------|------------|
| `skills/refine-us/` | Socratic discovery session. One grounded question at a time, A/B/C options, appends a `Technical Refinement` section to the story. |
| `skills/us-to-specs/` | Slices the refined story into vertical **tracer-bullet** specs under `.specs/`, with templates and an implementation-order file. |
| `skills/implement-specs-nestjs/` | Implements the specs one by one on a NestJS project: TDD, related tests while iterating, full build/test/lint per spec, one commit per spec, resumable. |
| `agents/development360.md` | A subagent that orchestrates the three stages and can resume a half-finished `.specs/` folder. |

## Install

**As a plugin (recommended)** — add this repository as a marketplace and install:

```
/plugin marketplace add <your-github-user>/development360
/plugin install development360
```

**Or copy the skills** into `~/.claude/skills/` (user-wide) or `<project>/.claude/skills/`
(project-only). Each skill is a folder with a `SKILL.md`; the `us-to-specs` templates must
travel with it.

## Workflow

### 1. Refine — `/refine-us docs/TICKET-123.md`

Reads the story *and* the codebase first, then asks only about what the story leaves open —
one question per message, with concrete options and a recommended default. Ends with a
summary; on your confirmation it appends:

```markdown
## Technical Refinement (2026-01-15)
### <Topic>
- <decision> — <why>
### Open questions
1. <question> — proposed default: <default>
```

Add `--quick` to batch the low-impact questions into one message.

### 2. Slice — `/us-to-specs docs/TICKET-123.md`

Proposes 4–8 vertical slices as a table (type AFK/HITL, blocked-by, acceptance criteria
covered), asks three quiz questions, and on approval writes:

```
.specs/TICKET-123-kebab-title/
├── 01-first-slice.md
├── 02-second-slice.md
└── implementation-order.md
```

### 3. Implement — `/implement-specs-nestjs .specs/TICKET-123-kebab-title/implementation-order.md`

Creates `feat/TICKET-123-kebab-title`, then for each spec: break down → tracer bullet →
TDD → `build` + full `test` + `lint` → record (`Agent summary`, order file) → commit
`TICKET-123 <description>`. Resumes from **Completed specs**. Options: `--spec NN`,
`--from NN`, `--no-branch`, `--no-commit`. Ends with `<promise>COMPLETE</promise>`.

This skill has `disable-model-invocation: true` — only you can start it.

## Conventions the skills assume (all overridable by your `CLAUDE.md`)

- Stories live in `docs/<TICKET>.md`; specs in `.specs/<TICKET>-<kebab-title>/`.
- Commit messages: `<TICKET> <description>`; branches: `feat/<TICKET>-<kebab-title>`.
- `package.json` exposes `build`, `test`, `lint` (and optionally a narrower `test:related`).
- Acceptance criteria are referenced as `AC1 … ACn` in order of appearance.

Your project's `CLAUDE.md` and path-scoped rules always take precedence.

## Adapting to another stack

`refine-us` and `us-to-specs` are stack-agnostic. To support another backend or a frontend,
copy `skills/implement-specs-nestjs/` to `skills/implement-specs-<stack>/`, rename it in the
frontmatter, and adjust the **Preflight** toolchain detection and the TDD paragraph.

## License

MIT — see `LICENSE`.
