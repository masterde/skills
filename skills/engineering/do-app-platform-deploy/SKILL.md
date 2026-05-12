---
name: do-app-platform-deploy
description: Deploy a marketing or studio-level Nuxt surface (e.g., xyz.dev-style site) to DigitalOcean App Platform with .do/app.yaml, env config, build commands, and custom domain attachment. The XYZ default for static-ish / mostly-static surfaces — not SaaS apps (those go to Fly.io). Use when deploying to DigitalOcean, configuring app.yaml, or saying "DO app platform", "digitalocean deploy", "ship marketing site", "deploy the studio site".
---

# Deploy to DigitalOcean App Platform

The XYZ default for **marketing / studio surfaces** — the kind of site where content changes more often than logic. `xyz.dev` runs here. For SaaS apps with auth, db writes, and frequent server logic, use `flyio-nuxt-deploy` instead.

## Why App Platform for these

- Zero-config builds for static + SSR Nuxt
- Built-in CDN, free TLS, painless domain attachment
- Auto-deploys from a GitHub branch with no separate CI
- Free tier is generous for marketing/studio sites
- Less surface than Fly for ops to maintain when no DB is involved

## Prerequisites

- `doctl` installed (`brew install doctl`)
- `doctl auth init` completed (uses a DO Personal Access Token)
- The site builds locally (`pnpm build`)
- GitHub repo connected to your DO account (one-time, via DO console)

## Process

### 1. App spec (`.do/app.yaml`)

Commit this to the repo so the deploy is reproducible.

```yaml
name: <site-name>
region: fra  # Frankfurt is closest to Bahrain in DO's PoPs

services:
  - name: web
    github:
      repo: masterde/<repo-name>
      branch: main
      deploy_on_push: true
    build_command: pnpm install --frozen-lockfile && pnpm build
    run_command: node .output/server/index.mjs
    environment_slug: node-js
    instance_count: 1
    instance_size_slug: basic-xxs
    http_port: 3000
    routes:
      - path: /
    envs:
      - key: NODE_ENV
        value: production
      - key: NITRO_PRESET
        value: node-server
      - key: NUXT_PUBLIC_SITE_URL
        value: https://xyz.dev
        scope: RUN_TIME

domains:
  - domain: xyz.dev
    type: PRIMARY
    zone: xyz.dev
```

**Notes on the defaults:**
- `region: fra` — Frankfurt is the closest DO PoP to Bahrain (low-latency for the team and EU users). DO has no GCC region as of 2026-05.
- `basic-xxs` — $5/month tier; fine for marketing/studio sites that don't see SaaS load.
- `deploy_on_push: true` — pushes to `main` auto-deploy. Production-safe because this is the studio's own content; for SaaS, prefer a separate staging branch.

### 2. Static-only sites (Nuxt generate)

If the site is pure static (`nuxi generate`, no SSR), use a `static_sites` block instead of `services`:

```yaml
static_sites:
  - name: web
    github:
      repo: masterde/<repo-name>
      branch: main
      deploy_on_push: true
    build_command: pnpm install --frozen-lockfile && pnpm generate
    output_dir: .output/public
    environment_slug: node-js
```

Static sites on App Platform are **free** (within bandwidth limits). Prefer this for any purely-marketing surface.

### 3. Secrets

For env vars that shouldn't be in `app.yaml`:

```bash
doctl apps update <app-id> --spec .do/app.yaml
doctl apps config set <app-id> SECRET_KEY=value --type SECRET
```

Or set them in the DO console under the app's Settings → Environment Variables (mark `Encrypted`).

### 4. Custom domain

Two paths:

**Path A — domain registered on Cloudflare/Namecheap/elsewhere:**
1. Add the domain in DO console (App → Settings → Domains) or via `app.yaml` as shown.
2. DO gives you DNS records (CNAME for subdomain, A/AAAA or apex-aliasing for root).
3. Set those records at the registrar.
4. DO issues a Let's Encrypt cert automatically. Wait 5–15min.

**Path B — domain on DO Networking:**
1. Add the domain at DO Networking → Domains.
2. Add the app domain in `app.yaml` with `type: PRIMARY` and `zone: <domain>`.
3. DO wires DNS automatically; cert issues in minutes.

For `xyz.dev` and XYZ's other TLDs, decide per-domain whether to use DO Networking or keep DNS with the existing registrar.

### 5. First deploy

```bash
doctl apps create --spec .do/app.yaml
```

Or via console: New App → "From Source Code" → pick the repo and branch.

For subsequent deploys, push to the configured branch — `deploy_on_push: true` handles the rest.

### 6. Promote / rollback

```bash
doctl apps list-deployments <app-id>
doctl apps create-deployment <app-id> --force-rebuild  # force redeploy
```

DO retains the last several deployments; rollback via console (Deployments tab → "Rollback").

## Verification

- `curl -I https://<domain>` returns 200 with `server: cloudflare` (DO uses CF as edge for App Platform)
- `doctl apps get <app-id>` shows `Phase: ACTIVE`
- Build logs in console show no errors
- Site loads at the custom domain with valid TLS

## When NOT to use this skill

- Anything with auth + DB writes (SaaS surface) → use `flyio-nuxt-deploy`.
- Apps that need WebSockets, background workers, or long-running connections → App Platform's defaults aren't tuned for these; Fly.io is better.
- Apps where every request hits Postgres → put compute near the DB (Fly Postgres in `bah` region).

## Reference

- XYZ topology: studio site (`xyz.dev`) on DO App Platform; SaaS products on Fly.io.
- DO App Platform pricing is per-instance and per-static-site; static sites are free within the bandwidth tier — confirm before scaling marketing infrastructure.
