# development360

A [Claude Code](https://docs.claude.com/en/docs/claude-code) plugin for **specs-driven
development**: take a user story from a vague idea to implemented, tested code in three
explicit stages, with a human decision point between each.

```
story.md ──▶ /refine-us ──▶ /us-to-specs ──▶ /implement-specs-<stack> ──▶ commits
             (decide)        (slice)           (typecheck · build · test · lint)

<stack> = nestjs | nextjs | react-native
```

## What's inside

| Path | What it is |
|------|------------|
| `skills/refine-us/` | Socratic discovery session. One grounded question at a time, A/B/C options, appends a `Technical Refinement` section to the story. |
| `skills/us-to-specs/` | Slices the refined story into vertical **tracer-bullet** specs under `.specs/`, with templates and an implementation-order file. |
| `skills/implement-specs-nestjs/` | Implements the specs one by one on a NestJS project: TDD, related tests while iterating, full build/test/lint per spec, one commit per spec, resumable. |
| `skills/implement-specs-nextjs/` | Same protocol for React / Next.js (App Router): Server Components by default, React Testing Library + MSW, optional Playwright e2e, `typecheck` → `lint` → `test` → `next build`, plus a senior checklist (a11y, UX states, performance, security, docs) walked per spec. |
| `skills/implement-specs-react-native/` | Same protocol for React Native (Expo or bare): screen-by-screen tracer bullets, React Native Testing Library with the project's test wrapper, native modules mocked at a wrapper boundary, optional Maestro/Detox e2e, native builds kept out of the edit loop, plus a senior checklist (a11y, safe areas/keyboard, offline, lists, permissions, secrets, docs). |
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

### 3. Implement — `/implement-specs-<stack> .specs/TICKET-123-kebab-title/implementation-order.md`

Pick the skill for your stack:

| Stack | Command | Verify step per spec |
|-------|---------|----------------------|
| NestJS | `/implement-specs-nestjs` | `build` → full `test` → `lint` |
| React / Next.js | `/implement-specs-nextjs` | `typecheck` → `lint` → full `test` → `next build` (→ `test:e2e` with `--e2e`) |
| React Native | `/implement-specs-react-native` | `typecheck` → `lint` → full `test` → project's bundle/native gate if any (→ `test:e2e` with `--e2e`) |

All three share the same protocol. Creates `feat/TICKET-123-kebab-title`, then for each
spec: break down → tracer bullet → TDD → verify → record (`Agent summary`, order file) →
commit `TICKET-123 <description>`. Resumes from **Completed specs**. Common options:
`--spec NN`, `--from NN`, `--no-branch`, `--no-commit`. Frontend skills add `--e2e`, and
React Native adds `--platform ios|android`. Ends with `<promise>COMPLETE</promise>`.

The frontend skills ship a `references/senior-checklist.md` (architecture, TypeScript,
accessibility, performance, UX states, security, testing bar, documentation) that the
skill reads once in preflight and walks for every file touched before a spec is called
done. The project's own `CLAUDE.md` and lint config always win over the checklist.

These skills have `disable-model-invocation: true` — only you can start them.

## Conventions the skills assume (all overridable by your `CLAUDE.md`)

- Stories live in `docs/<TICKET>.md`; specs in `.specs/<TICKET>-<kebab-title>/`.
- Commit messages: `<TICKET> <description>`; branches: `feat/<TICKET>-<kebab-title>`.
- `package.json` exposes `build`, `test`, `lint` (frontends: also `typecheck`, optionally `test:e2e`, and a narrower `test:related`).
- Acceptance criteria are referenced as `AC1 … ACn` in order of appearance.

Your project's `CLAUDE.md` and path-scoped rules always take precedence.

## Adapting to another stack

`refine-us` and `us-to-specs` are stack-agnostic. To support another stack, copy the closest
`skills/implement-specs-<stack>/` folder, rename it in the frontmatter, adjust the
**Preflight** toolchain detection, the testing rules in step 4, the verify order in step 5,
and rewrite `references/senior-checklist.md` for that stack. Then add the detection rule to
`agents/development360.md` and the hand-off in `skills/us-to-specs/SKILL.md`.

## License

MIT — see `LICENSE`.
