# CLAUDE.md — panpawka/shelve fork

Personal fork of [HugoRCD/shelve](https://github.com/HugoRCD/shelve), self-hosted on Vercel.
This file and `vercel.json` are the only intended diffs against upstream. Everything else
should track `upstream/main`.

## Instance

- Canonical URL: **https://shelve-ochre.vercel.app** (Vercel team `lemonode`, project `shelve`)
- Vercel root directory: `apps/shelve`; build via root `vercel.json` (`nuxt prepare --cwd ../base && nuxt build`)
- Deploys via `vercel deploy --prod` from repo root — no git integration
- Database: Neon Postgres (external, survives deploys). Connection via `DATABASE_URL` env on Vercel
- Instance data: team slug `lemonode`, projects `Feednode` and `PersonalBeats`,
  environments `{development,staging,production}-{client,server}`

## Future syncs with upstream

1. `git fetch upstream`
2. `git branch -f backup/pre-sync-<date> main` (push it) — cheap rollback point
3. `git reset --hard upstream/main`
4. `git checkout <previous main> -- vercel.json CLAUDE.md` and commit
5. Check new migrations under `apps/shelve/server/db/migrations/postgresql/` for destructive
   statements (TRUNCATE/DROP) before anything runs them — see "Migrations" below
6. `pg_dump` the Neon DB first if migrations touch data
7. `git push --force-with-lease origin main`
8. `vercel deploy --prod`
9. Verify: app serves 200, `POST /api/auth/device` returns a code, `shelve pull` works

Do not merge upstream into main — fork-only commits accumulate conflicts. Reset + reapply
the two files above.

## Migrations — read before installing with prod env

NuxtHub (`@nuxthub/core`, `hub.db: 'postgresql'`) applies pending SQL migrations
**automatically**, including during the `nuxt prepare` postinstall of `pnpm install`,
whenever `DATABASE_URL` is set. Journal table: `_hub_migrations` in the Neon DB.
Consequence: `pnpm install` with production env sourced = production schema migration.
Dump the DB before doing that.

## Env vars

All required env lives on Vercel (`vercel env pull <file> --environment=production`):
GitHub OAuth pair, `NUXT_SESSION_PASSWORD`, `NUXT_PRIVATE_ENCRYPTION_KEY`, `DATABASE_URL`.
`NUXT_PRIVATE_ENCRYPTION_KEY` is the KEK for stored variables — losing or changing it makes
every stored secret undecryptable. Redis vars on Vercel are legacy; the app no longer uses Redis.

## Gotchas

- **GitHub OAuth only works on the canonical URL.** The OAuth app (`Ov23liqHyq3SOxPO1dtK`)
  has `https://shelve-ochre.vercel.app/auth/github` as its callback. Per-deployment URLs
  (`shelve-<hash>-lemonode.vercel.app`) fail with "redirect_uri is not associated".
- Shelve v5 (2026-08 sync) hashed all API tokens and truncated the `tokens` table by design;
  pre-v5 tokens cannot be migrated. CLI auth is a browser device flow now (`shelve login`).
- CLI version should match the deployed server; both come from this repo's release line
  (`@shelve/cli` on npm). Login failures after a long gap usually mean version skew — sync
  and redeploy instead of pinning an old CLI.
- The repo-root `shelve.json` points at `http://localhost:3000` — that is upstream's dev
  default, leave it. Per-project CLI config lives in each consuming project's own `shelve.json`.
