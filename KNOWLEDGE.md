# ExpiresSoon — architecture and operations

Engineering context for this repo: the demo model, the status engine, hosting, CI,
and the gotchas that cost time. Setup and commands live in `README.md` and
`AGENTS.md`; this file is the "why" and the "watch out".

## 1. What it is

Track things that expire (licences, subscriptions, documents, insurance,
warranties). Items carry title, category, expiration date, notes. Status
(`active` / `expiring_soon` / `expired`) is **derived per request**, never stored.

It ships as an anonymous live demo: no accounts, each visitor gets a private
ephemeral room, plus a Tally.so waitlist.

## 2. Layout

```
artifacts/api-server/     Express 5 API          @workspace/api-server
artifacts/expiry-tracker/ React 19 + Vite SPA    @workspace/expires-soon  (legacy folder name)
lib/api-spec/             openapi.yaml — source of truth
lib/api-zod/              generated Zod schemas
lib/api-client-react/     generated TanStack Query hooks
lib/db/                   Drizzle schema + client   @workspace/db
```

## 3. The anonymous demo (core model)

Each visitor gets a **private room** keyed by a UUID in an `HttpOnly`,
`SameSite=Lax`, **no-expiry** cookie (`expiresoon_demo`). Closing the browser ends
the room. The cookie is `secure` only in production.

The cookie is **signed**: `<uuid>.<base64url HMAC-SHA256>`, verified in constant
time. `ownerId` is the tenancy key for every row, so an unsigned cookie would let
anyone read, edit, or reset another room.

- **Production requires `SESSION_SECRET`** or the API refuses to boot
  (`openssl rand -base64 32`; set it in Render's environment). Dev/test use a fixed
  insecure fallback.
- Changing the secret invalidates every room. Give each environment its own.

Request lifecycle: `requireSession` -> `ensureSession` (verify or mint the cookie)
-> sets `req.ownerId` -> `next()`. No database work, no seeding.

### Seeding and onboarding (opt-in only)

Rooms are **not** seeded automatically, so a cold visitor costs zero writes (the
free-tier compute limit is the constraint, not storage).

The empty room shows a 3-step interactive tour
(`components/first-run/first-run-empty.tsx`): pick a preset -> watch the status
badge change -> add your first item. Dismissal is stored in `sessionStorage`
(`expiresoon_tutorial_skipped`) with a replay link.

The 4-item sample roster lives in `artifacts/api-server/src/seed/` and is written
only by `POST /api/session/reset`. Its button renders only on an empty room, so it
cannot destroy real items. Seeding is idempotent: an unlocked pre-check, then a
transaction-scoped `pg_advisory_xact_lock(hashtext(ownerId))` plus re-check.

### Limits (all in `config/index.ts`, documented in `.env.example`)

| Limit | Default |
|---|---|
| Items per room (`MAX_ITEMS_PER_OWNER`) -> 409 | 10 |
| Requests/IP/window across `/api` | 300 |
| Session resets/IP/window | 10 |
| Item creations/IP/window | 60 |

The limiter is **in-process**: behind N replicas the effective limit is xN. It keys
on `req.ip`, which is correct only because `app.ts` sets `trust proxy` to exactly
`1` — add another proxy hop and every client collapses into one bucket (and a
spoofed `X-Forwarded-For` becomes possible).

## 4. Status engine

`lib/status.ts`: `computeStatus(date)` / `enrichItem(row)`.

- `expired` — before today · `expiring_soon` — within `EXPIRING_SOON_DAYS` (30) ·
  `active` — otherwise.
- `EXPIRING_THIS_WEEK_DAYS` (7) drives the dashboard "this week" bucket.
- Uses UTC calendar midnights (`Date.UTC` + `Math.round`) so server timezone and
  DST cannot shift the day count.
- When building a Date from `YYYY-MM-DD`, append `T00:00:00` to force local-time
  interpretation.

## 5. Input validation (`lib/validation.ts`)

- `expiration_date` must be a real `YYYY-MM-DD` date; `isValidDateOnly` rejects
  impossible dates like `2026-02-31` that `Date.parse` would silently roll over.
- Hand-written, not `format: date` in the spec: Orval is configured with
  `useDates: true` + `coerce.body`, so `format: date` would generate
  `zod.coerce.date()` and hand the repository a `Date` instead of the string its
  column wants.
- It fails closed on purpose: `computeStatus("banana")` is `NaN`, and every
  comparison with `NaN` is false, so the item would be reported `active`.
- `escapeLikePattern` escapes `\`, `%`, and `_` so search matches literally.

## 6. Hosting

- Web: static Vite build on Netlify (`netlify.toml`, publish
  `artifacts/expiry-tracker/dist/public`, Node 24).
- API: Express on Render.
- DB: Neon serverless Postgres.

`artifacts/expiry-tracker/public/_redirects` proxies `/api/*` to the Render host —
**if API calls fail, check this file still points at the live Render URL.**

```
/api/*    https://expiresoon-api.onrender.com/api/:splat   200
/*        /index.html   200
```

- `lib/db/src/index.ts` fails fast if `DATABASE_URL` is missing and sets
  connection/query/idle timeouts so a dead DB errors instead of hanging.
- TLS verification for the DB is **on** by default (`lib/db/src/ssl.ts`); opt out
  only with `DATABASE_SSL_REJECT_UNAUTHORIZED=false`.
- Index: `items_owner_id_expiration_date_idx` on `(owner_id, expiration_date)`.
  The `search` `ILIKE '%term%'` path cannot use it (it would need `pg_trgm`), so
  search is bounded by the per-room cap and the client debounce.
- `GET /healthz` is served at the root **and** `/api/healthz`; the root route sits
  outside the rate limiter so frequent probes don't count against it.
- `index.ts` handles `SIGTERM`/`SIGINT`: drain, close the pool, exit 0 (10s cap,
  idempotent). Render sends `SIGTERM` on every deploy.

## 7. CI

`.github/workflows/ci.yml`, on push to `main` and on PRs. `typecheck-and-test`
(ubuntu-24.04, ephemeral Postgres 16, `RUN_DB_TESTS=1`): checkout -> setup-node 24
-> corepack pnpm 10.30.3 -> install -> typecheck -> `db push` -> Vitest.
Do **not** add `cache: pnpm` to setup-node — it cannot find pnpm before corepack
provides it.

`deploy-netlify` runs only after that passes and only on `main`.
`nwtgck/actions-netlify` exits 0 without deploying if `NETLIFY_AUTH_TOKEN` /
`NETLIFY_SITE_ID` are missing, so a green check does **not** prove the site updated.

## 8. Codegen

`lib/api-spec/openapi.yaml` -> Orval -> `lib/api-zod` + `lib/api-client-react`.
Never hand-edit the generated files; change the YAML and run
`pnpm --filter @workspace/api-spec run codegen`.

Known drift: the spec documents the reset endpoint as `POST /session`, but the
server implements `POST /session/reset`, so the generated reset hook is unused.

## 9. Local development gotchas

- No Vite proxy — set `VITE_API_BASE_URL`.
- `.env` is not auto-loaded (no dotenv dependency); pass vars via the shell.
- The API `dev` script is bash-only and fails in PowerShell.
- Both the API (`PORT || 5000`) and Vite (`PORT || 3000`) read `PORT` — don't
  export the same value in both terminals.

## 10. Product and UI notes

- 204: `DELETE /api/items/:id` returns 204 with an empty body; `parseResponse`
  special-cases 204/205. Do not add an unconditional `response.json()` — the
  delete toast regresses.
- Client retry: `custom-fetch.ts` retries once, only on a quick network failure
  (Render cold boot), never after a consumed timeout and never on HTTP errors.
- Filters live in the URL; `useItemFilters` re-syncs `status` from the query string
  (wouter matches pathname only, so a query change does not remount the list).
- Search is debounced 300 ms and LIKE-escaped server-side, because each request is
  an unindexed `%term%` scan.
- Design tokens in `src/index.css`: explicit radius tiers (`sm` 6 -> `xl` 20 ->
  `2xl` 24) so shape encodes hierarchy. `corner-shape: squircle` is progressive
  enhancement (Chromium 139+ only) — never ship a design that *depends* on it, and
  keep `corner-shape: inherit` on the `hover-elevate` overlays.
- Motion respects `prefers-reduced-motion: reduce` (a block at the end of
  `index.css`), with `.animate-spin` exempt. `framer-motion` is a dependency but is
  never imported.
- Fonts and the Tally embed are self-hosted / lazily injected to avoid leaking
  visitor IPs to third parties. Do not reintroduce a font CDN or a static Tally
  `<script>`. See the compliance section in `ROADMAP.md`.

## 11. Archived

`docs/archive/` holds finished point-in-time records (CI incident log, delete-button
investigation, older implementation plans). They may describe superseded states —
read them as history.
