# Contributing to ExpiresSoon

Issues and pull requests are welcome. Small, single-purpose changes are strongly
preferred — one concern per PR, easy to review, easy to revert.

## Before you open a PR

1. `pnpm install`
2. `pnpm run typecheck` — must pass. This is the only automated gate; there is no
   linter or formatter configured.
3. `pnpm --filter @workspace/api-server test` — run the API tests.
   DB-backed suites (`*.owner.test.ts`) need `RUN_DB_TESTS=1` and a real
   `DATABASE_URL`; otherwise they skip silently, so a green run is not proof.
4. If you changed the API contract, run
   `pnpm --filter @workspace/api-spec run codegen` and commit the regenerated files.

## Rules of the repo

- Never hand-edit `lib/api-zod/**` or `lib/api-client-react/src/generated/**` —
  they are Orval output. Edit `lib/api-spec/openapi.yaml` and regenerate.
- Expiry status is never stored; it is derived. Thresholds live in
  `artifacts/api-server/src/config/index.ts` — never hard-code them.
- Item access goes through `IItemsRepository` and is always scoped by `ownerId`.
- One concern per PR. State your verification steps in the description.

## Read first

- `README.md` — setup, testing, deployment, API.
- `AGENTS.md` — commands, package names, gotchas.
- `KNOWLEDGE.md` — architecture and operations.
- `decision documentation.md` — why things are the way they are.

The [MIT licence](LICENSE) applies to all contributions.
