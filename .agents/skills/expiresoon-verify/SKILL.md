---
name: expiresoon-verify
description: Verify an ExpiresSoon change before declaring it done.
license: MIT
metadata:
  author: omar
  version: "0.1.0"
  project: ExpiresSoon
---

# ExpiresSoon — verification runbook

Use before declaring ANY ExpiresSoon code change finished, and when CI is red.
`AGENTS.md` (repo root) holds the portable rules and commands; this skill holds the
gate, the expected failures, and what to do about them.

## The gate (run from the repo root)

1. `pnpm run typecheck` — **the primary verification. There is no linter or
   formatter.** It runs `tsc --build tsconfig.json` (builds `lib/db`,
   `lib/api-zod`, `lib/api-client-react` project references) then typechecks every
   package. Expect exit 0 and no output.
2. `pnpm --filter @workspace/api-server test` — Vitest. DB-backed suites
   (`*.owner.test.ts`) run only when `RUN_DB_TESTS=1` **and** a real `DATABASE_URL`
   is set; otherwise they skip silently — a green run does NOT mean they ran.
3. If the change touches the data model: apply it with
   `pnpm --filter @workspace/db run push`. There are no migration files; `push`
   mutates Postgres directly, so it is destructive-by-design — never run it
   against production to "check" a schema change.
4. If the OpenAPI contract changed: `pnpm --filter @workspace/api-spec run codegen`
   to regenerate Zod schemas and TanStack Query hooks. The script's final step
   calls an undefined root script and errors — that error is EXPECTED; run
   `pnpm run typecheck` afterwards instead of debugging it.

Never hand-edit generated files: `lib/api-zod/**` and
`lib/api-client-react/src/generated/**` are Orval output; `lib/api-spec/openapi.yaml`
is the only hand-written contract source.

## Expected failures → cause → fix

- **TS6305 (`…dist/index.d.ts has not been built`) + a wall of TS7006
  implicit-any** → a referenced lib was never built, so TS fell back to `any`.
  Fix by running the root `tsc --build tsconfig.json` (i.e. `pnpm run typecheck`),
  not by adding type annotations. `--build` must be the FIRST CLI arg.
- **TS2345 `SQL | undefined not assignable to SQL`** → Drizzle `or(...)` returns
  `SQL | undefined`; guard it before pushing into a `conditions` array
  (`if (filter) conditions.push(filter)`).
- **TS2322 `AbortSignal | null` not assignable** → `RequestInit.signal` may be
  `null`; use a truthiness check, not `typeof === "undefined"`.

## Invariants a change must not break

- Expiry **status is never stored**; it is derived at request time in
  `artifacts/api-server/src/lib/status.ts`. Never add a status column or cache it.
- Thresholds (`EXPIRING_SOON_DAYS`, `EXPIRING_THIS_WEEK_DAYS`) live in
  `artifacts/api-server/src/config/index.ts`. Never hard-code 30/7 anywhere else.
- All item access goes through `IItemsRepository`; add methods to the interface
  first, then the Drizzle impl. Every query is scoped by `ownerId`.
- `SESSION_SECRET` is required in production (the API refuses to boot without it);
  dev/test fall back to a fixed insecure secret.

## Deploy / CI reality

- CI (`.github/workflows/ci.yml`): typecheck → `drizzle-kit push` → Vitest, then a
  Netlify deploy on `main`. A red typecheck blocks the deploy.
- Do NOT add `cache: pnpm` to `setup-node` — it fails before corepack provides pnpm.
- SPA deploys to Netlify; API to Render; Netlify proxies `/api/*` via
  `artifacts/expiry-tracker/public/_redirects`.
- The dev API `dev` script uses bash `export` and fails in PowerShell — build then
  start with `NODE_ENV=development` explicitly.

## Before proposing UI/UX work

Read `ROADMAP.md` first — it lists approaches already **rejected with reasons**
(⛔ items) plus done/planned steps. Check the current dashboard: PR #11
(`feat/overview-redesign`) rewrote `pages/dashboard.tsx` and
`components/spotlight-action.tsx`, so older notes may be stale.
