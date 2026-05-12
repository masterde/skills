---
name: new-xyz-venture
description: End-to-end checklist for spinning up a new XYZ venture — name/domain selection from the studio's TLD portfolio, GitHub repo creation, hosting choice (Fly.io for SaaS, DO App Platform for marketing), monorepo placement, base routes, and the bootstrap sequence. Use when starting a new XYZ product, saying "new venture", "spin up a saas", "kick off a new product", "start an xyz product".
---

# Spin up a new XYZ venture

The 9-step checklist for taking a venture from "we should build X" to "X has a repo, a domain, a deploy pipeline, and a running localhost." Designed for venture-builder velocity — not over-engineered upfront.

## Step 1 — Confirm the venture is real

Before any code:

- One sentence: what does this product do?
- One sentence: who is the target customer?
- One sentence: how does it make money?
- Decision: is this a **standalone SaaS** (own repo, own domain, own DB) or a **module of an existing XYZ product** (lives inside an existing repo)?

If the user can't answer all four, stop and grill them first (use the `grill-me` skill).

## Step 2 — Name + domain

**Naming conventions for XYZ products:**
- Repo: `xyz-<short-name>` on GitHub (org: `masterde` for now, until XYZ org is set up)
- Product name: short, pronounceable, ideally with an available `.com` or `.co`
- Don't reuse a name that conflicts with existing ventures (Dealmatter, Orderly, RuleModel, WhatsApp AI Receptionist)

**Domain — pick from XYZ's portfolio when it fits:**

| TLD | Best for |
|---|---|
| `xyz.dev` | The studio itself — taken |
| `xyz.ventures` | Portfolio / public-facing ventures hub |
| `xyz.management` | Internal ops / management SaaS for the studio |
| `xyz.marketing` | Marketing-tool products |
| `xyz.click` | URL shorteners, tracking, redirect-heavy products |
| `xyz.help` | Support / docs / helpdesk SaaS |
| `xyz.team` | Team / recruiting / internal portal |
| `xyz.sa` | KSA-targeted products (Saudi Arabia ccTLD) |
| `xyz.onl` | Catchall / short alias |

If none fits, the venture may justify buying a fresh domain (e.g., `dealmatter.co`, `orderly.now`). Use a new TLD only when the product brand is meaningfully independent from the XYZ studio brand.

## Step 3 — Hosting choice

| Workload | Host | Skill to invoke |
|---|---|---|
| SaaS with auth + DB + frequent writes | **Fly.io** | `flyio-nuxt-deploy` |
| Marketing / mostly-static / studio-level surface | **DigitalOcean App Platform** | `do-app-platform-deploy` |

Default for new SaaS ventures: Fly.io. Default for any `xyz.*` marketing surface: DO App Platform.

## Step 4 — Create the GitHub repo

```bash
gh repo create masterde/xyz-<short-name> --private --description "<one-line product summary>"
gh repo clone masterde/xyz-<short-name>
cd xyz-<short-name>
```

Private by default. Open-source only when there's a clear reason.

## Step 5 — Scaffold the stack

Invoke the `xyz-nuxt-saas-scaffold` skill for the canonical Nuxt 4 + Prisma 7 + Better Auth stack. That skill handles:
- Nuxt init
- Prisma 7 with `prisma-client` generator + `@prisma/adapter-pg`
- Better Auth wiring (with organization plugin if multi-tenant)
- Nuxt UI v4 + Tailwind
- Initial `AGENTS.md` / `CLAUDE.md`

After scaffold runs, the venture has a working `pnpm dev` at `localhost:3000`.

## Step 6 — Wire deploy

Invoke `flyio-nuxt-deploy` (or `do-app-platform-deploy` for marketing). This produces:
- `fly.toml` / `.do/app.yaml` committed
- Postgres attached (Fly Postgres or Neon)
- Secrets set
- First successful deploy to `<venture-name>.fly.dev` or DO equivalent
- Custom domain pointed (DNS configured, certs issued)

## Step 7 — Add base routes

Every XYZ SaaS gets these on day one (so it never has to backfill them under pressure later):

- `/` — landing page (placeholder OK)
- `/sign-in`, `/sign-up`, `/forgot-password` — Better Auth flows
- `/app/*` — authenticated area (protected by Better Auth middleware)
- `/api/health` — returns `{ ok: true }` for uptime monitors
- `/api/auth/*` — Better Auth handler
- `/legal/privacy` and `/legal/terms` — placeholder; mandatory for any payment integration later
- `/admin` — gated to org owner (when org plugin is on)

For GCC-targeted products, also invoke the `rtl-arabic-nuxt` skill on day one — retrofitting RTL is painful.

## Step 8 — Register in the studio index

Update XYZ's central project list (location TBD — likely a Notion page or `xyz-management` venture):
- Venture name
- Domain
- Repo URL
- Host
- One-line summary
- Current status (alpha / beta / live)

## Step 9 — Update memory

Add a one-line entry for this new venture to the `project_xyz_ventures.md` memory file at `/Users/baker/.claude/projects/-Users-baker-Projects-skills/memory/project_xyz_ventures.md`. Future Claude sessions need to know this venture exists.

## Verification

- Repo exists on GitHub, is private, has the `xyz-<name>` prefix
- `pnpm dev` boots at `localhost:3000` and shows the landing page
- `pnpm dlx prisma migrate dev` runs clean against the dev DB
- A sign-up flow creates a user in the Better Auth tables
- The deploy command succeeds; the custom domain serves the app with TLS
- `/api/health` returns 200 from production

## When NOT to use this skill

- Adding a feature to an existing venture — that's not a "new venture."
- Throwaway experiments — use `prototype` instead.
- Internal tools that don't justify a domain — those can live as routes inside `xyz-management`.
