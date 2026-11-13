# ExpiresSoon

Track things that expire — licences, subscriptions, documents, insurance, warranties.
Each item has a title, category, expiration date, and notes. Status (*Active* /
*Expiring Soon* / *Expired*) is computed on every request, never stored.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

## What it does

- Full CRUD over tracked items.
- Status derived at query time: expired (< today), expiring soon (<= 30 days, configurable), active otherwise.
- Dashboard with status counts, a "needs attention" queue, and an "expiring this week" spotlight.
- Debounced search (title/category) and URL-synced status tabs.
- 3-step interactive first-run tour on an empty room.
- Light/dark theme, mobile-first layout.
- OpenAPI-first: one spec, generated client hooks and Zod schemas.
- Anonymous demo: an HMAC-signed cookie maps each visitor to a private, ephemeral room. No accounts.
- Early-access waitlist via a Tally.so modal.

## Stack

| Layer | Choice |
|---|---|
| Frontend | React 19, Vite 7, Tailwind 4, Wouter, TanStack Query v5 |
| Forms | React Hook Form + Zod |
| API | Express 5 on Node 24 (ESM), Pino logging |
| Database | PostgreSQL + Drizzle ORM |
| Contract | OpenAPI 3.1 |
| Monorepo | pnpm workspaces |

## Run it

Requires Node 24+, pnpm 10.30.3 (via Corepack), PostgreSQL 15+.

```bash
git clone https://github.com/omar-m-h18/ExpiresSoon
cd ExpiresSoon
pnpm install
pnpm --filter @workspace/db run push
```

Config is read from `process.env` (there is no dotenv). See `.env.example`. Minimum:

```bash
export DATABASE_URL="postgresql://postgres:password@localhost:5432/expiresoon"
export PORT=3001
```

Two terminals:

```bash
# API
pnpm --filter @workspace/api-server run build
NODE_ENV=development pnpm --filter @workspace/api-server run start

# Web (port 3000)
export VITE_API_BASE_URL="http://localhost:3001"   # required: there is no Vite dev proxy
pnpm --filter @workspace/expires-soon run dev
```

On Windows/PowerShell the API `dev` script fails (it uses bash `export`) — build
then `start`, setting `$env:NODE_ENV="development"` first.

## Verify

```bash
pnpm run typecheck                                # primary gate; there is no linter or formatter
pnpm --filter @workspace/api-server test          # Vitest
RUN_DB_TESTS=1 DATABASE_URL="postgresql://..." \
  pnpm --filter @workspace/api-server test        # DB-backed suites; without this they skip
```

## Deploy

- Web: static Vite build on Netlify (`netlify.toml`); `public/_redirects` proxies `/api/*` to the API so there is a single origin.
- API: Express on Render.
- DB: Neon serverless Postgres.
- CI (`.github/workflows/ci.yml`): typecheck -> drizzle push -> Vitest, then a Netlify deploy on `main` only.

## API

Base path `/api`. Contract: `lib/api-spec/openapi.yaml`.

| Method | Path | Notes |
|---|---|---|
| GET | `/healthz` | Also served at root `/healthz`. |
| GET | `/session` | Validates the signed cookie. |
| POST | `/session/reset` | Clears the room, writes 4 sample items. |
| GET | `/items` | Filters: `search`, `status`, `sort`. |
| POST | `/items` | Capped at `MAX_ITEMS_PER_OWNER` (default 10) -> 409 when full. |
| GET | `/items/summary` | Status counters. |
| GET/PATCH/DELETE | `/items/:id` | DELETE returns 204. |

Regenerate the client and Zod schemas:
`pnpm --filter @workspace/api-spec run codegen`.

## Layout

```
ExpiresSoon/
├── artifacts/
│   ├── api-server/       # Express 5 API         (@workspace/api-server)
│   └── expiry-tracker/   # React + Vite SPA      (@workspace/expires-soon)   <- legacy folder name
├── lib/
│   ├── api-spec/         # openapi.yaml (source of truth)
│   ├── api-zod/          # generated Zod schemas
│   ├── api-client-react/ # generated TanStack Query hooks
│   └── db/               # Drizzle client + schema
├── docs/archive/         # finished point-in-time records
├── AGENTS.md             # commands + gotchas for coding agents
├── KNOWLEDGE.md          # architecture and operations
├── ROADMAP.md            # planned work + rejected approaches
└── decision documentation.md   # ADR log
```

## Read next

- `AGENTS.md` — repo commands, package names, gotchas (start here if you are an agent).
- `KNOWLEDGE.md` — architecture, demo model, hosting, CI.
- `ROADMAP.md` — planned work and deliberately rejected ideas.
- `CONTRIBUTING.md` — how to contribute.

## License

MIT (c) ExpiresSoon Contributors
