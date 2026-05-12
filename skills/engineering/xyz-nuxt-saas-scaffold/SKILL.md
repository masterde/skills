---
name: xyz-nuxt-saas-scaffold
description: Scaffold a new XYZ SaaS on the canonical XYZ stack — Nuxt 4 + Prisma 7 (prisma-client generator + @prisma/adapter-pg) + Better Auth + Tailwind/Nuxt UI v4 + pnpm. Use when starting a new venture, bootstrapping a Nuxt SaaS, or saying "new XYZ SaaS", "scaffold venture", "bootstrap nuxt saas", "new product on the XYZ stack".
---

# Scaffold an XYZ SaaS

Bootstrap a new venture on XYZ's canonical stack. Don't deviate from this stack without an explicit reason — the leverage compounds across XYZ's portfolio when every venture looks the same.

## Canonical XYZ stack

| Layer | Choice | Notes |
|---|---|---|
| Framework | **Nuxt 4** | `compatibilityVersion: 4` set in `nuxt.config.ts` |
| Package manager | **pnpm** | with workspace if part of a monorepo |
| UI | **Nuxt UI v4** + Tailwind CSS | Reka UI under the hood for headless primitives |
| ORM | **Prisma 7** | `prisma-client` generator (NOT the legacy `prisma-client-js`) + `@prisma/adapter-pg` driver adapter |
| Database | **Postgres** | Neon for dev/staging; provider for prod confirmed per-venture |
| Auth | **Better Auth** | with organization plugin if the SaaS is multi-tenant |
| State | Pinia (only if needed) | prefer composables first |
| Validation | Zod | shared between server routes and forms |
| Content | Nuxt Content v3 | only if marketing/docs pages live in-app |

## Process

### 1. Explore

Before scaffolding, check:
- `git remote -v` — is this a fresh repo or being added to an existing one?
- `package.json` if present — what's already in place?
- Whether this is a standalone repo or a workspace inside a monorepo (look for `pnpm-workspace.yaml`, parent-dir `package.json`)
- The venture's intended host (Fly.io for SaaS by default — see `flyio-nuxt-deploy`)

### 2. Confirm decisions one-by-one

Walk the user through these **in order**, get an answer before moving on. Don't dump them all at once.

**A — Venture name and domain.** Which of XYZ's TLDs does this product use? (xyz.dev, .ventures, .management, .marketing, .click, .help, .team, .sa, .onl, or a new TLD purchase.) Record both the product name and the target domain.

**B — Multi-tenant or single-tenant?** Affects Better Auth setup (organization plugin yes/no) and Prisma schema (per-tenant scoping).

**C — Marketing site in-app or separate?** If marketing pages need Nuxt Content v3, scaffold with content; otherwise keep the app focused.

**D — Monorepo or standalone?** If joining the XYZ monorepo (when one exists), use the workspace conventions; if standalone, the skill scaffolds a self-contained repo.

### 3. Scaffold

Run, in order:

```bash
pnpm dlx nuxi@latest init <venture-name>
cd <venture-name>
```

Then add dependencies. Use `pnpm add` (not `npm` / `yarn`):

```bash
pnpm add @nuxt/ui @nuxt/content
pnpm add prisma @prisma/client @prisma/adapter-pg
pnpm add better-auth @onmax/nuxt-better-auth
pnpm add -D @types/node
```

For Prisma 7, use the **`prisma-client` generator** (replaces `prisma-client-js` in v7). Initialize schema with the canonical XYZ pattern — see the `prisma-nuxt-setup` skill (globally available) for the full pattern: split schema across `prisma/schema/*.prisma`, singleton client in `server/utils/db/`, typed query helpers.

```prisma
// prisma/schema/schema.prisma
generator client {
  provider = "prisma-client"
  output   = "../node_modules/.prisma/client"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}
```

### 4. Wire Better Auth

Use `@onmax/nuxt-better-auth` (the Nuxt module wrapper). If multi-tenant, enable the organization plugin per the `organization-best-practices` skill (globally available).

`server/utils/auth.ts`:
```ts
import { betterAuth } from "better-auth";
import { prismaAdapter } from "better-auth/adapters/prisma";
import { organization } from "better-auth/plugins";
import { prisma } from "./db";

export const auth = betterAuth({
  database: prismaAdapter(prisma, { provider: "postgresql" }),
  emailAndPassword: { enabled: true },
  plugins: [organization()],
});
```

### 5. Add `nuxt.config.ts` essentials

```ts
export default defineNuxtConfig({
  compatibilityDate: "2026-01-01",
  future: { compatibilityVersion: 4 },
  modules: [
    "@nuxt/ui",
    "@onmax/nuxt-better-auth",
    // "@nuxt/content", // if marketing/docs in-app
  ],
  runtimeConfig: {
    databaseUrl: "",
    betterAuthSecret: "",
    public: {
      siteUrl: "",
    },
  },
});
```

### 6. Add an `AGENTS.md` / `CLAUDE.md`

Record the stack choice, the venture's domain, the host (Fly.io / DO), and a one-line summary of what the product does. This is what `setup-xyz-skills` reads later.

### 7. Final checks

- `pnpm install` runs clean
- `pnpm dlx prisma migrate dev --name init` produces an initial migration
- `pnpm dev` boots without errors at `http://localhost:3000`
- Better Auth's `/api/auth/get-session` returns null for an anonymous user

## Reference skills to read

These globally available skills carry the deep patterns this skill orchestrates:

- `prisma-nuxt-setup` — the canonical DB layer for Nuxt 4
- `better-auth-best-practices` — auth config, session management, plugins
- `nuxt-better-auth` — the `@onmax/nuxt-better-auth` module specifically
- `organization-best-practices` — multi-tenant org/team/role/invite flows
- `nuxt-ui` — Nuxt UI v4 component patterns

## When NOT to use this skill

- Adding features to an existing XYZ SaaS — those are venture-specific edits, not scaffolding.
- Marketing-only sites — use `do-app-platform-deploy` for static / mostly-static surfaces deployed to DigitalOcean App Platform.
- Prototypes meant to be thrown away — use the existing `prototype` skill.
