# Decision documentation

A record of the decisions taken in this repository, why, and what they affect.

Rules: one section per decision, numbers are never reused; a reversed decision adds
a new entry that points back (nothing is deleted); entries marked **Proposed** are
not settled — do not build on them.

---

## D-001 — Anonymous rooms no longer auto-seed sample data
**2026-10-04 · Accepted — implemented 2026-10-05 on `feat/empty-room-onboarding`**

Every new room got 8 sample rows from `generateSampleItems()`, awaited inside
`requireSession` before `next()`. That burned free-tier Neon compute on visitors who
never returned, and it hid the activation cost of an empty screen.

**Decision:** rooms start empty; `requireSession` is synchronous and does no DB work.
The empty state is explained by the UI instead.

**Affects:** `middlewares/requireSession.ts`, `seed/`, `first-run-empty.tsx`,
`pages/dashboard.tsx`, `pages/items-list.tsx`.

---

## D-002 — No feature flag for seeding
**2026-10-05 · Accepted**

**Decision:** no `SEED_SAMPLE_DATA` flag. Automatic seeding is removed
unconditionally; the only remaining seeding call site is user-triggered.

**Why:** a flag is a way to forget. If the examples button were ever wired up while
the flag stayed off, it would clear the visitor's room and then show nothing — silent
data loss for the person who clicked. One unconditional rule cannot be misconfigured,
and "should a cold visitor cost eight writes?" has only one defensible answer on a
free tier.

**Affects:** no new env var; `config/index.ts` unchanged.

---

## D-003 — Seeding is a self-contained module
**2026-10-04 · Accepted — implemented 2026-10-05**

Seeding was spread across `lib/seed.ts`, `lib/sample-data.ts`, and inline in
`requireSession`.

**Decision:** move it to `artifacts/api-server/src/seed/`, one file per
responsibility, one public entry point — `date-offset.ts` (pure `dayOffsetISO`),
`sample-items.ts` (roster + `generateSampleItems`), `index.ts` (only public surface:
`seedRoomIfEmpty`, `clearRoom`), `sample-items.test.ts`. Nothing outside the folder
imports `sample-items` or `date-offset`.

**Why:** this folder is exactly what will **not** port to a self-hosted build;
isolating it makes that boundary visible (see D-008).

---

## D-004 — The delete-button bug is a stale bundle, not a logic bug
**2026-10-04 · Accepted — fix in source, deploy was the blocker**

The delete button showed "Failed to delete item" while the network tab showed 204 and
the row was gone from the DB. The live bundle's fetch wrapper ends with
`return await response.json()` with no 204/205 guard; on an empty body that throws, so
TanStack Query calls `onError` and `onSuccess`/`invalidateQueries` never runs.

The fix already exists in source: `parseResponse` in
`lib/api-client-react/src/custom-fetch.ts` returns early on 204/205 (landed
2026-08-29, commit `2bdca69`). The live bundle predated it.

The 204 also proves this was *not* a room/cookie mismatch — that produces 404.

**Decision:** no logic change. Ship the existing fix by repairing the deploy pipeline.

**Affects:** `.github/workflows/ci.yml`; see `docs/archive/DELETE_BUTTON_INVESTIGATION.md`.

---

## D-005 — CI must prove the deploy reached the live site
**2026-10-04 · Accepted — implemented 2026-10-05**

`nwtgck/actions-netlify@v3` exits 0 when its credentials are missing
(`fails-without-credentials` defaults to false), so a green check could ship nothing —
which is how a months-old bundle stayed live.

**Decision:** three guards, in order: (1) `fails-without-credentials: true`;
(2) a build-time tripwire that fails if the output references the known-stale bundle
hash `index-C2cMtcLG.js`; (3) a post-deploy step comparing the bundle filename in the
build output against the live site. Guard 3 is the one that matters — "is the site
up?" is useless because the stale bundle also answers yes.

Both jobs are pinned to `ubuntu-24.04` (specific versions usually get runner capacity
sooner than the moving label).

---

## D-006 — One concern per change
**2026-10-04 · Accepted**

Large mixed commits are hard to review, revert, and bisect.

**Decision:** every change ships as its own small PR with a single purpose. No bundled
refactors, no drive-by formatting. If two things must change together, the split is
wrong. Each PR states its verification steps and can be reverted alone.

---

## D-007 — Normalize line endings first
**2026-10-04 · Done — commit `12805e4`**

`core.autocrlf=true` made the working tree report ~198 modified files while
`git diff --stat` reported zero changed lines.

**Decision:** add `.gitattributes` with `* text=auto eol=lf`, set `core.autocrlf false`,
then `git add --renormalize .` in a commit containing nothing else.

**Verification:** `git diff HEAD~1 --stat --ignore-cr-at-eol` returns only
`.gitattributes`.

---

## D-008 — Self-hosting comes later
**2026-10-04 · Accepted as direction, not designed**

The product tracks certificates, insurance, licences, and ID documents — data visitors
expect to keep on their own machine. A hosted demo is a good funnel; a self-hosted
install is the better final answer.

**Decision:** do not build it now. When the time comes, decide early whether it is the
same codebase behind a flag or a thin wrapper.

**Ports:** schema, `lib/status.ts`, the thresholds, the UI.
**Does not port:** the anonymous room, the signed session cookie, every argument about
Neon quota (which is why D-003 isolates the seed module).

---

## D-009 — The tutorial is three steps, not a manual
**2026-10-04 · Accepted — implemented 2026-10-05**

Removing seeding (D-001) makes the first screen a set of zeroes; something must close
that gap.

**Decision:** the first-run screen gives three short steps and one button that ends in
the visitor's own first item; the examples button sits quietly below as the other
choice. **Why:** a visitor from the landing page wants to see the product work, not
read documentation.

---

## D-010 — Release on a branch, merge once
**2026-10-05 · Accepted**

**Decision:** all work happens on a feature branch; the owner merges to `main` when
satisfied, giving one API deploy and one web deploy together.

**Why:** the API auto-deploys from `main` through Render's git integration, so a push
to `main` cannot be held back — a branch is the only way to release several changes
together. A branch push runs `typecheck-and-test` only (the deploy job is gated on
`github.ref == 'refs/heads/main'`).

---

## D-011 — The examples button only appears on an empty room
**2026-10-05 · Accepted**

`POST /api/session/reset` clears the caller's room and writes the example roster — it
is destructive by design.

**Decision:** the button renders only when the server reports the room holds zero
items. Emptiness comes from the room-wide `total` in the summary response, not the
filtered list, so a search or status tab can never make a populated room look empty.
If the summary request fails, the room is reported **not** empty, so an error never
replaces real content with an onboarding prompt.

**Affects:** `hooks/use-room-is-empty.ts` (shared by both pages so they cannot disagree).

---

## D-012 — Room cleanup is a separate, later job
**2026-10-05 · Accepted as direction, not built**

Nothing ever deletes a room.

**Decision:** do not build cleanup now. After the tutorial, add a scheduled job
(Render cron) that deletes rooms untouched for a long time.

**Why:** two different costs are being confused. *Writing* rows burns compute, the
limit that matters on a free tier, and D-001 stops it. *Storing* old rows is far
cheaper. Cleanup is worthwhile but not urgent, and it is the riskiest change to get
wrong — a bug there deletes real visitors' data rather than demo noise.

---

## D-013 — Tutorial not showing: two causes
**2026-10-07 · Accepted — CI fix implemented; deploy unblocked by a manual Netlify publish**

Either cause was sufficient on its own.

1. **Wrong bundle live.** The site kept serving the August `index-C2cMtcLG.js`; the new
   bundle containing `FirstRunEmpty` was uploaded but never promoted, because Netlify
   **Auto publishing** is disabled (deliberate — keeping build minutes at zero). The
   workflow is: Actions builds and uploads -> owner clicks "Publish deploy". That step
   was missed.
2. **Existing seeded rooms.** `useRoomIsEmpty` reads `summary.total`; visitors who
   arrived before D-001 still have 8 rows, so `FirstRunEmpty` never renders for them.
   Not a bug — they already saw the product. Cleanup is D-012.

**CI fix:** `nwtgck/actions-netlify` was writing commit comments/statuses with a
`GITHUB_TOKEN` that only has `contents: read` + `metadata: read`, causing three 403s.
Those cosmetic features are now off. A 10s sleep was added before the bundle comparison
so the Netlify edge CDN can flush.

---

## D-014 — Room capped at 10 items, sample roster reduced to 4
**2026-10-08 · Accepted — implemented on `feat/empty-room-onboarding`**

**Decision:**

1. **Cap:** `MAX_ITEMS_PER_OWNER` default 100 -> 10 (anonymous ephemeral rooms holding
   100 rows risk table bloat on a free Neon tier).
2. **Roster:** trim `SAMPLE_SPECS` to 4 (Netflix active, SSL certificate expiring soon,
   Car Insurance expiring this week, Passport expired) — covers every status card and
   leaves 6 slots open. An 8-item roster would have consumed 80% of the new cap.
3. **Feedback:** `demo-banner.tsx` shows `X / 10 items` and switches to a destructive
   style at the cap; `item-form.tsx` disables submit at the limit.

---

## D-015 — Waitlist delegated to the Tally.so modal
**2026-10-08 · Accepted — implemented on `feat/empty-room-onboarding`**

An inline form posting to `POST /api/leads` added maintenance (storage, rate limiting,
no anti-spam or qualification).

**Decision:** delegate to Tally.so (form `lbPjoV`): popup via
`src/lib/tally.ts` (width 540, 4s timeout, falls back to a new tab), placed on the
landing page and the app sidebar. Backend cleanup removed `POST /api/leads`,
`leads.repository.ts`, `RATE_LIMIT_MAX_LEADS`, and `demo.ts`, so no waitlist request
ever writes to Neon.

---

## D-016 — Production review: DST-safe date math, deterministic sort, form race
**2026-10-08 · Accepted — implemented on `feat/empty-room-onboarding` and `main`**

**Decision:**

1. **DST/timezone immunity:** `computeStatus` uses pure UTC calendar timestamps
   (`Date.UTC`) with `Math.round`, so local timezone and DST boundaries cannot shift the
   day count. (It previously used `Math.floor` on local-midnight Dates, which is wrong
   on 23- and 25-hour days.)
2. **Deterministic ordering:** add `asc(itemsTable.id)` as a tie-breaker in `orderBy`;
   sorting on `expiration_date` alone returned ties in non-deterministic disk order.
3. **Form state:** guard `isSubmitDisabled` with `(isNew && isLoadingSummary)` so the
   submit button never flashes enabled before capacity is known.

---

## D-017 — Interactive, skippable onboarding walkthrough
**2026-10-08 · Accepted — implemented on `feat/empty-room-onboarding` and `main`**

**Decision:** turn `FirstRunEmpty` into an interactive stepper — (1) clickable item
presets (*Passport*, *Car Insurance*, *Domain Name*, *Streaming*) that update a
simulated live card; (2) an interactive status explorer (*Active*, *Expiring Soon*,
*Expired*) explaining non-stale background calculation; (3) a choice between loading
the 4 samples and creating a custom item. Add a "Skip tutorial" action persisted in
`sessionStorage` (`expiresoon_tutorial_skipped`) with a replay link.

---

## D-018 — Early-access elevation, calendar layout, category chips
**2026-10-08 · Accepted — implemented on `feat/empty-room-onboarding` and `main`**

**Decision:**

1. **Elevate early access across 5 touchpoints:** top demo banner pill, sidebar footer
   callout card, mobile tab-bar action, landing hero, and a dismissible milestone
   banner in the items list.
2. **Calendar:** enforce `w-[280px]` root width with `w-9` weekday headers and
   `h-9 w-9` day buttons — without it `react-day-picker` collapses to `min-content`
   and squishes `SuMoTuWeThFrSa`.
3. **Category chips:** quick-pick pills (*Documents*, *Subscriptions*, *Insurance*,
   *Software*, *Warranties*, *Health*) under the category input.
4. **Navigation:** move "Add Item" into the primary sidebar nav, removing the duplicate
   bottom button.

---

## D-019 — Onboarding simplification and temporary sample-button concealment
**2026-10-08 · Accepted — implemented on `feat/empty-room-onboarding` and `main`**

**Decision:**

1. **Conceal the sample button:** `SHOW_SAMPLE_BUTTON = false` in `first-run-empty.tsx`
   (the handler and `useResetSession` stay in code for later re-enablement) — the button
   was unreliable in the live deployment.
2. **Hands-on tour:** step 1 guided preset choice with visual checkmarks; step 2 date
   timeframe buttons (*In 1 Year*, *In 2 Weeks*, *5 Days Ago*) that update the card and
   status badge live; step 3 a single "Add your first item" CTA plus three takeaways.

---

## Rejected and superseded

- **`SEED_SAMPLE_DATA` feature flag** — superseded by D-002. A flag is a way to forget,
  and the failure mode is silent data loss for the person who clicks.
- **A local test runner in `lib/api-client-react`** — rejected for now: adding a
  dependency changes `package.json` without the matching lockfile entry, which breaks
  CI's `pnpm install`. The bundle-comparison guard (D-005) is the regression protection.
- **Shipping the seeding removal alone** — rejected in favour of D-010, because a push
  to `main` cannot be held back and would have shipped empty rooms with no guidance.
