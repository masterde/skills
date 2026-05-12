---
name: xyz-venture-handoff
description: Onboard a Claude agent (or human) to a specific XYZ venture — which codebase, host, domain, current state, shared packages in use, and what to read first. Builds on the generic `handoff` skill with XYZ-specific structure. Use when switching contexts between ventures, saying "switch to <venture>", "context for <venture>", "onboard me to <product>", or "load <venture> context".
argument-hint: "Which venture? (Dealmatter / Orderly / RuleModel / WhatsApp AI Receptionist / xyz.dev / other)"
---

# XYZ venture handoff

Generate a context briefing for an agent (or human) starting work on a specific XYZ venture. This is the venture-builder cousin of the generic `handoff` skill — it captures studio-level facts the generic version doesn't know about.

## Process

### 1. Identify the venture

If the user passed an argument (`Dealmatter`, `Orderly`, `RuleModel`, `WhatsApp AI Receptionist`, or `xyz.dev`), use it. Otherwise ask which venture.

If the venture name doesn't match a known one, check `/Users/baker/.claude/projects/-Users-baker-Projects-skills/memory/project_xyz_ventures.md` to see if it's been added since this skill was last updated.

### 2. Gather the venture's facts

For the named venture, locate (or ask for):

- **Repo location** — local path and GitHub URL
- **Domain** — production domain(s)
- **Host** — Fly.io or DigitalOcean App Platform
- **Database** — Fly Postgres, Neon, or other; staging vs prod connection
- **Branch in flight** — what's currently being worked on
- **Open PRs / issues** — pending review or in progress
- **Recent activity** — `git log --oneline -10` on the venture's repo
- **Shared packages in use** — any `@xyz/*` packages from the studio's shared layer

If working in a Claude Code session inside the venture's repo, gather facts directly. If not, ask the user for the repo path so the skill can `cd` and read.

### 3. Write the handoff doc

Save it to `$(mktemp -t xyz-handoff-XXXXXX.md)` (read the file before writing). Structure:

```markdown
# XYZ handoff — <venture name>

## Studio context
- Studio: XYZ (venture-builder, Bahrain)
- This venture's role in the portfolio: <one line>

## Venture facts
- Repo: `<github url>` (local: `<path>`)
- Domain(s): <list>
- Host: <Fly.io | DO App Platform>
- DB: <Fly Postgres | Neon | other>; connection scoping
- Stack: Nuxt 4 + Prisma 7 + Better Auth (or deviations from canonical)

## Current state
- Branch in flight: `<branch>`
- Last commit: `<sha>` — `<message>`
- Open PRs: <list with URLs>
- Open issues being worked: <list>

## Live entry points
- Production: <url>
- Staging: <url or N/A>
- Local dev: `pnpm dev` → http://localhost:3000

## Shared packages
- <list `@xyz/*` packages this venture imports, or "none yet">

## What to read first
- `AGENTS.md` / `CLAUDE.md` in the repo root
- `CONTEXT.md` if present
- The handoff this session is replacing (path, if any)

## Suggested skills for this session
- <skills that match the venture's current work>

## Open questions for the next agent
- <anything ambiguous, blocked, or needing decisions>
```

### 4. Tell the user

Print the path to the saved handoff doc. Suggest which skills the next session is likely to need (e.g., `tdd` if writing tests, `diagnose` if hunting a bug, `rtl-arabic-nuxt` if working on GCC UI).

## Notes

- Do not duplicate content already in PRDs, plans, ADRs, issues, or commits — link to them by path or URL.
- Cross-venture work (extracting shared `@xyz/*` packages, multi-venture refactors) needs a different handoff format; flag and ask instead of forcing the structure.
- If the venture isn't yet in memory, run `new-xyz-venture` step 9 (memory update) afterward.

## When NOT to use this skill

- Within-venture handoffs (mid-feature, same project) — use the generic `handoff` skill; it's lighter.
- A user asking "what is XYZ?" — that's a studio-level question, answer from memory directly.
