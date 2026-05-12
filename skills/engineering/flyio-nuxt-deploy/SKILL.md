---
name: flyio-nuxt-deploy
description: Deploy a Nuxt SaaS to Fly.io with fly.toml, Postgres attachment (Fly Postgres or Neon), Prisma client provisioning during build, and env config. The canonical XYZ path for all SaaS products after migrating off Vercel due to Prisma WASM runtime issues. Use when deploying to Fly.io, setting up fly.toml, configuring flyctl, or saying "deploy to fly", "fly.io nuxt", "ship to fly".
---

# Deploy a Nuxt SaaS to Fly.io

The canonical XYZ deploy target for SaaS products. Fly.io was chosen after Vercel's serverless runtime broke Prisma's new `prisma-client` generator (WASM-based query engine). Fly's Firecracker microVMs run the same code as local dev.

## Prerequisites

- `flyctl` installed (`brew install flyctl`)
- `fly auth login` completed (one-time)
- Postgres provider chosen — Fly Postgres or external (Neon recommended for dev; Fly Postgres or managed elsewhere for prod)
- The Nuxt app builds clean locally (`pnpm build`)

## Process

### 1. Initialize Fly app

```bash
fly launch --no-deploy --copy-config=false
```

When prompted:
- **App name:** `<venture-name>` (must be globally unique on Fly)
- **Region:** start with `bah` (Bahrain) for AST-co-located workloads, or `fra` (Frankfurt) / `dxb` (Dubai) for nearest-EU/Gulf datacenters
- **Postgres / Redis:** decline at this step; attach separately so we control the config

This creates a `fly.toml`. Replace it with the XYZ template below.

### 2. Canonical `fly.toml`

```toml
app = "<venture-name>"
primary_region = "bah"

[build]
  builder = "paketobuildpacks/builder:base"
  buildpacks = ["gcr.io/paketo-buildpacks/nodejs"]

[env]
  NODE_ENV = "production"
  NITRO_PRESET = "node-server"
  PORT = "3000"

[http_service]
  internal_port = 3000
  force_https = true
  auto_stop_machines = "stop"
  auto_start_machines = true
  min_machines_running = 0
  processes = ["app"]

  [http_service.concurrency]
    type = "requests"
    hard_limit = 250
    soft_limit = 200

[[vm]]
  size = "shared-cpu-1x"
  memory = "512mb"
```

**Why these defaults:**
- `primary_region = "bah"` — co-locates compute with the team's timezone and the most likely GCC user base. Adjust per product.
- `auto_stop_machines + min_machines_running = 0` — keeps idle SaaS products free or cheap; first request after idle has a ~1s cold start (acceptable for B2B).
- `shared-cpu-1x 512mb` — fine for small Nuxt apps; scale via `fly scale memory 1024` if Prisma + Nitro pushes past 400MB at idle.

### 3. Dockerfile (when buildpack isn't enough)

Buildpacks work for vanilla Nuxt. If the venture needs Prisma migrations during deploy or has native dependencies, use a Dockerfile:

```dockerfile
FROM node:20-alpine AS base
RUN corepack enable

FROM base AS deps
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile

FROM base AS builder
WORKDIR /app
COPY . .
COPY --from=deps /app/node_modules ./node_modules
RUN pnpm dlx prisma generate
RUN pnpm build

FROM base AS runner
WORKDIR /app
ENV NODE_ENV=production
COPY --from=builder /app/.output ./.output
COPY --from=builder /app/node_modules/.prisma ./node_modules/.prisma
COPY --from=builder /app/node_modules/@prisma ./node_modules/@prisma
EXPOSE 3000
CMD ["node", ".output/server/index.mjs"]
```

Then remove the `[build]` block from `fly.toml` and set `[build] dockerfile = "Dockerfile"`.

**Critical for Prisma 7:** the `prisma-client` generator outputs to `node_modules/.prisma/client` by default. The Dockerfile must copy this directory into the runner stage, or queries fail at runtime with cryptic engine errors. This is the gotcha that drove the migration off Vercel.

### 4. Postgres attachment

**Option A — Fly Postgres (managed cluster on Fly):**
```bash
fly postgres create --name <venture-name>-db --region bah
fly postgres attach <venture-name>-db --app <venture-name>
```
This sets `DATABASE_URL` automatically.

**Option B — Neon (external):**
```bash
fly secrets set DATABASE_URL="postgresql://..." --app <venture-name>
```
Use Neon's pooled connection string. Confirm `@prisma/adapter-pg` is wired so pooled connections work cleanly.

### 5. Secrets

```bash
fly secrets set \
  BETTER_AUTH_SECRET="$(openssl rand -base64 32)" \
  NUXT_PUBLIC_SITE_URL="https://<venture-name>.fly.dev" \
  --app <venture-name>
```

Never commit secrets; never set them via `[env]` in `fly.toml`.

### 6. Run migrations on deploy

Add a `release_command` to `fly.toml`:

```toml
[deploy]
  release_command = "pnpm dlx prisma migrate deploy"
```

Fly runs `release_command` in a one-off machine before flipping traffic — safe place to run migrations.

### 7. Deploy

```bash
fly deploy
```

Watch logs:
```bash
fly logs --app <venture-name>
```

### 8. Custom domain

```bash
fly certs add <product-domain> --app <venture-name>
```

Fly prints DNS records to configure. Point the domain (one of XYZ's TLDs) via the domain registrar, then `fly certs check <product-domain>` until status is `Ready`.

## Verification

- `fly status` shows machines `started`
- `curl https://<venture-name>.fly.dev/api/health` returns 200 (if a health route exists; add one if not)
- Prisma query from a route works (no "PrismaClient initialization failed" or WASM engine errors)
- `fly logs` shows no startup errors
- Better Auth session endpoint responds

## When NOT to use this skill

- Marketing / static-ish sites → use `do-app-platform-deploy` for DigitalOcean App Platform instead.
- The studio's own site `xyz.dev` → DO App Platform.
- Anything that needs Vercel Edge Functions specifically → reconsider the requirement; XYZ has explicitly moved off Vercel.

## Reference

- The migration off Vercel was driven by Prisma WASM runtime issues. If a project tries the `prisma-client-js` (legacy) generator + Vercel combo and works, that's not a path forward — XYZ's stack mandates Prisma 7's new `prisma-client` generator.
- For local dev parity, install `flyctl` and use `fly proxy 5432 -a <venture-name>-db` to connect to Fly Postgres from local Prisma Studio.
