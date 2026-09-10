# ShipAny Cloudflare Development Guide

This document is the working agreement for people and coding agents contributing to this repository.

## Project Purpose

This repository is the Cloudflare-oriented ShipAny Template Two application. It is the **single source of truth** for ongoing secondary development and Cloudflare deployment.

- Canonical project: `/Users/wender/Desktop/www/shipany-template-cf`
- Do **not** make feature changes in `/Users/wender/Desktop/www/shipany-template-dev` and copy them here afterward.
- The `-dev` directory uses a different runtime/database setup (including SQLite) and can drift from Cloudflare behavior.

## Stack and Runtime Model

- Framework: Next.js `15.5.7` with App Router and Turbopack in development.
- Cloudflare adapter: OpenNext for Cloudflare (`@opennextjs/cloudflare`).
- Database: PostgreSQL via Drizzle ORM.
- Local database: PostgreSQL 16 in Docker Compose.
- Production target: Cloudflare Workers with a managed PostgreSQL database; Cloudflare Hyperdrive is the intended production connection mechanism.

### Local Architecture

```text
Browser → Next.js dev server (localhost:3000) → PostgreSQL Docker container (localhost:5432)
```

Use normal Next.js development for daily work, then verify Worker compatibility before deploying.

## Local Setup

### One-time setup

1. Install JavaScript dependencies using the repository's intended package manager:

   ```bash
   pnpm install
   ```

   This repository has `pnpm-lock.yaml`. Do not routinely mix `npm install` and `pnpm install`, because they produce different lockfiles.

2. If `pnpm` is unavailable in a terminal, enable it using Corepack:

   ```bash
   corepack enable
   corepack prepare pnpm@11.19.0 --activate
   pnpm --version
   ```

   `npm run dev` is acceptable for launching the existing `dev` script when pnpm is unavailable, but use pnpm for dependency-management operations where possible.

3. Ensure `.env.development` exists. It is intentionally ignored by Git. A local configuration should contain at least:

   ```env
   NEXT_PUBLIC_APP_URL="http://localhost:3000"
   NEXT_PUBLIC_APP_NAME="ShipAny App"
   NEXT_PUBLIC_THEME="default"
   NEXT_PUBLIC_APPEARANCE="system"

   DATABASE_PROVIDER="postgresql"
   DATABASE_URL="postgresql://shipany:shipany_dev_password@127.0.0.1:5432/shipany"
   DB_SINGLETON_ENABLED="true"
   DB_MAX_CONNECTIONS="1"

   AUTH_SECRET="<generate with openssl rand -base64 32>"
   ```

### Start the local environment

```bash
# Start PostgreSQL and preserve existing database data.
docker compose -f compose.dev.yml up -d

# Start Next.js.
pnpm dev
# or, if pnpm is not available in the current terminal:
npm run dev
```

Open `http://localhost:3000`.

Check database health:

```bash
docker compose -f compose.dev.yml ps
```

Stop PostgreSQL while preserving data:

```bash
docker compose -f compose.dev.yml down
```

**Destructive:** `docker compose -f compose.dev.yml down -v` deletes the local PostgreSQL volume. Do not run it unless a database reset is explicitly intended.

## Database Rules

- Keep this project on PostgreSQL locally and in production. Do not switch it to the SQLite configuration from the separate `shipany-template-dev` directory.
- PostgreSQL schema export is controlled by `src/config/db/schema.ts`.
- After schema changes, run one of the following with Docker PostgreSQL running:

  ```bash
  pnpm db:generate
  pnpm db:migrate
  ```

  For fast local schema synchronization during early development:

  ```bash
  pnpm db:push
  ```

- Initialize RBAC only when a fresh development database requires it:

  ```bash
  pnpm rbac:init
  ```

- Never commit `.env*`, database dumps, real credentials, tokens, or generated migration directories that are intentionally ignored by the repository.

## Development Workflow

1. Make focused changes in this repository.
2. Run the affected feature locally with `pnpm dev` (or `npm run dev`).
3. For schema changes, update PostgreSQL with a migration or `db:push`.
4. Before handing off a feature, validate production compilation:

   ```bash
   pnpm build
   ```

5. Before a Cloudflare deployment or when adding server-side dependencies, validate the Worker build:

   ```bash
   pnpm cf:preview
   ```

6. Deploy only after explicit approval and after Cloudflare settings/secrets are configured:

   ```bash
   pnpm cf:deploy
   ```

## Cloudflare Deployment Rules

- `open-next.config.ts` and Cloudflare-specific code are required deployment infrastructure; do not remove or replace them with the files from `shipany-template-dev`.
- Copy `wrangler.toml.example` to `wrangler.toml` for a personal/local deployment configuration. `wrangler.toml` is ignored by Git.
- Production Worker secrets must be set with Wrangler, not committed in `[vars]`:

  ```bash
  npx wrangler secret put AUTH_SECRET
  ```

- Production database credentials should use Cloudflare Hyperdrive. Its binding name is `HYPERDRIVE` and must agree with `wrangler.toml` and `src/core/db/postgres.ts`.
- Do not use local filesystem persistence for uploads or application data in Cloudflare Workers. Use an object store such as Cloudflare R2/S3-compatible storage.

## Important Files

| Path | Responsibility |
| --- | --- |
| `compose.dev.yml` | Local PostgreSQL 16 service |
| `.env.development` | Local-only application/database secrets; ignored |
| `src/config/db/schema.ts` | Active Drizzle schema export |
| `src/core/db/` | Dialect-specific database connections and selection |
| `src/middleware.ts` | Request middleware for this Next.js/Cloudflare version |
| `next.config.mjs` | Next.js, MDX, i18n, and OpenNext dev integration |
| `open-next.config.ts` | OpenNext Cloudflare configuration |
| `wrangler.toml.example` | Cloudflare Worker/Hyperdrive configuration template |
| `src/app/` | Routes, pages, API handlers |
| `src/themes/` | Theme UI implementation |
| `src/config/locale/` | Internationalized message files |

## Constraints and Conventions

- Preserve compatibility with Cloudflare Workers. Avoid adding Node-only APIs (`fs`, `child_process`, raw TCP modules, etc.) to code that can execute in the Worker runtime.
- Do not upgrade Next.js, OpenNext, React, Drizzle, or Cloudflare packages as part of an unrelated feature. Dependency upgrades require a dedicated task and full local/Cloudflare validation.
- Keep locale changes synchronized across English and Chinese messages where the feature is user-facing.
- Prefer small, isolated commits/changes. State any new environment variables and migration requirements in the handoff summary.
- Do not expose or print secret values in logs, commits, comments, PR descriptions, or task summaries.

## Validation Notes

- `pnpm build` is the minimum required production compilation check.
- The template historically did not include a complete ESLint flat configuration. If `pnpm lint` fails due to missing configuration or pre-existing violations, report it separately; do not weaken lint rules merely to make a feature appear green.
- A successful normal Next.js build does not guarantee Cloudflare compatibility. Use `pnpm cf:preview` before release.

## Handoff Checklist

For every completed feature, report:

1. What changed and the files involved.
2. Database changes and commands run.
3. New/changed local or Cloudflare environment variables (without revealing values).
4. Commands run and their outcomes.
5. Any known limitations, follow-up work, or deployment actions the owner must take.
