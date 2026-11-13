# Roadmap

Forward-looking work: UI/UX for the SPA (`artifacts/expiry-tracker`) plus two
independent product items (first-run tour, RTL Arabic).

Legend: ✅ done · ⬜ planned · ⛔ rejected (see bottom)

---

## ⬜ Step 4 — Landing page: one idea at a time  (high)

**File:** `src/pages/landing.tsx` (only).

**Problem:** the page stacks seven ideas in one `max-w-2xl` column and asks for the
same action three times (three waitlist CTAs). The demo-notice card duplicates
`DemoBanner`.

**Plan:** five full-bleed bands, Problem -> Solution -> Proof -> CTA:

1. **Hero** — H1 + one sentence + one CTA ("Start the demo").
2. **"One date. That's the input."** — the add-item form as the visual.
3. **"It tells you what's urgent."** — the three real status states via `StatusBadge`.
4. **"Private by default."** — no signup, sandbox room, signed session.
5. **Early access** — the single remaining waitlist ask + footer.

Each band: full-bleed background, inner `max-w-6xl`, one heading, <= 30 words, one
visual, `min-h-[70vh]` on desktop.

**Acceptance:** exactly one primary CTA and one waitlist CTA; no band with more than
one idea; visuals reuse `Card`/`StatusBadge`/`Input` (there are no screenshots); the
Tally modal still opens.

---

## ⬜ Step 5 — Scroll-driven motion  (medium · depends on step 4)

All motion today is one-shot on mount; nothing scrolls, so nothing is
scroll-linked. `framer-motion` is a dependency but never imported.

**Plan:** a ~20-line `useReveal` hook (`IntersectionObserver`) that toggles the
existing `tw-animate-css` classes, plus a CSS-only status-card stack built from the
real `Card` + `StatusBadge` with a subtle scroll-linked tilt. 200–300 ms ease-out,
reveal once per session.

**Acceptance:** gated behind `prefers-reduced-motion`; no new runtime dependency; no
layout shift (bands reserve their space).

---

## ⬜ Step 6 — Dashboard density  (low · mostly done)

PR #11 rewrote `dashboard.tsx`: status tiles now carry share meters and % captions,
a staggered entrance, and a proper "Needs attention" section.

**What's left (optional):** decide whether `Total items` still deserves equal grid
weight now that it carries a status-mix bar; if not, demote it to a quiet header row.
**Non-goal:** do not add `rotateY`/`scale` scroll transforms to data surfaces — it
hurts scanability of dates and statuses.

---

## ⬜ Step 7 — First-run tour commits the first item  (high · ~half a day)

**File:** `src/components/first-run/first-run-empty.tsx`.

**Problem:** the tour teaches then discards — the user picks a preset and a timeline,
sees a real preview card, then step 3 sends them to an empty form.

**Plan:** make step 3 a **commit** step. Everything needed is already in component
state (`selectedPreset`, `selectedTimeline`) and `useCreateItem` exists. Show a
summary ("We will create **Passport** · Documents · expires 12 Mar 2027 ·
*Expiring soon*"), disable while pending, catch 409, then land on `/demo/items`.
Keep "I'll do it myself" and "Skip tour".

**Gotcha:** the API needs `YYYY-MM-DD` from local parts — do **not** use
`toISOString()` (timezone shift). And step 2 *promises* a status the server
*computes*: with defaults they agree, but lowering `EXPIRING_SOON_DAYS` would make
the tutorial lie — derive offsets from `config/index.ts` or document the coupling.

**Acceptance:** creates exactly one item and lands on `/demo/items`; the status
matches the step-2 badge; a double-click cannot create two; the written date is the
local calendar date (verify under `TZ=Pacific/Kiritimati`); both escape hatches work.

**Test matrix:** 4 presets x 3 timelines, plus the double-click and 409 paths.

---

## ⬜ Step 8 — Native RTL Arabic version  (medium · ~1 week + a native writer)

> ASD-STE100 is a controlled **English** standard — it does not apply to Arabic.
> Do not translate the STE100 rules. A separate Arabic style guide is needed.

**Current state:** no localization; `<html lang="en">` with no `dir`; all text
hardcoded English; DM Sans/Outfit have no Arabic glyphs. Direction work is small —
**32 direction-dependent utilities across 15 files** (`ml-` 3, `mr-` 9, `pl-` 4,
`left-` 3, `right-` 3, `text-left/right` 4, `translate-x` 6).

**Plan:**

1. **Direction** — set `lang`/`dir`; set `dir="rtl"` before first paint via an
   inline script in `index.html` (setting it in React flashes LTR first); swap
   physical for logical utilities (`ml-`->`ms-`, `pl-`->`ps-`, `left-`->`start-`,
   `text-left`->`text-start`); mirror directional icons. The `flex-row` sidebar
   moves right on its own.
2. **Font** — self-host IBM Plex Sans Arabic or Noto Sans Arabic (**not** a CDN —
   that reintroduces the GDPR problem). Never apply `letter-spacing`/`tracking-tight`
   to Arabic; raise line-height to ~1.7.
3. **Strings** — ~40 user-facing strings. Prefer a typed dictionary + `useLocale()`
   (`Intl.PluralRules` handles Arabic plurals) over adding `react-i18next`.
4. **Dates/plurals** — pass the active locale to `toLocaleDateString` and import the
   `date-fns` `ar` locale; Arabic has six plural categories (use
   `Intl.PluralRules("ar")`); pick one digit set; wrap Latin text in `<bdi>`.
5. **Server messages** — return stable error codes or map client-side; no English
   text in an Arabic UI.
6. **Content** — use a native Arabic writer (**do not machine-translate**); swap
   presets for local documents (*Iqama*, *Istimara*, passport, medical insurance);
   decide on Hijri dates; the Tally form is English-only.

**Acceptance:** RTL check passes on all 5 routes, light and dark; plurals correct
for 0/1/2/3–10/11–99/100; dates correct in the popover and list rows; Latin text
does not reorder; no LTR flash on reload; no English string reachable from the
Arabic UI.

---

## ✅ Done

- **Step 1** — dashboard build blocker: fixed by the Overview redesign (PR #11).
- **Step 2** — explicit radius tiers + progressive `corner-shape: squircle`.
- **Step 3** — `prefers-reduced-motion` guard.
- **Step 9** — renamed to **ExpiresSoon** (free on `.com`/`.io`/`.app`/npm and the
  GitHub/Netlify/Render slugs). A get-by name, not a trademark moat; no formal
  clearance search was run.

---

## ⛔ Deliberately rejected

| Idea | Why not |
|---|---|
| `border-radius: 24px 22px 20px 18px` | Not a squircle — reads as a rendering bug. Use `corner-shape` or a symmetric tier. |
| `snap-y snap-mandatory` on mobile | The page hosts a form and a Tally modal; mandatory snap traps touch scrolling and fights iOS address-bar resize. |
| Splitting the add-item form into screens | `item-form.tsx` is already correctly chunked; splitting adds friction. |
| Restructuring the 3-step tour | It already implements one idea per step — step 7 changes what step 3 *does*, not the structure. |
| A dashboard chart | Expiry data is a triage list, not a time series. `recharts` is a dependency but unused. |
| `UseBy` / `BestBefore` / `StillGood` as the name | Best semantic fits, but all registered on `.com`, `.io`, and `.app`. |
| Latin-derived names (`Perimo`, `Deficio`, `Evanesco`) | Read as pharmaceutical brands; hard to police. |
| `Lapsive` | Reads as "laxative". |

---

## Compliance follow-ups

Not UI work. Statutory figures are maximums; this is not legal advice — have counsel
review before launch.

### ✅ Fixed

- **Third-party font CDN (GDPR)** — DM Sans/Outfit/Inter are now self-hosted; the CDN
  link and `preconnect` hints were removed.
- **Tally embed loaded before consent** — the script is now injected on first
  waitlist click (`src/lib/tally.ts`), not statically in `index.html`.
- **No transparency at collection (Art. 13)** — a purpose + opt-out line was added
  under the landing CTA.
- **`notes` invited sensitive IDs (Art. 5(1)(c))** — the placeholder now asks for a
  renewal reference, with a "don't store passwords/cards/IDs" helper.

### ⬜ Open

- **`/api/leads` declared but unimplemented** — do not mount until the table has
  `consented_at`, `source`, and `unsubscribe_token`. The SPA uses Tally.
- **No privacy policy or terms page** — required before launch; needs the legal
  entity, contact address, and sub-processor list (Neon, Render, Netlify, Tally).
- **No unsubscribe path** — no email is sent yet; add a signed token, a one-click
  endpoint, and a postal address before the first send.
- **No retention or erasure** — rooms persist in Neon indefinitely. Add a "Delete my
  data" action and a scheduled sweep.
- **No processor agreements (Art. 28, Ch. V)** — all vendors are US-hosted; need DPAs
  plus SCCs or the Data Privacy Framework.

### ⛔ Not applicable

- Session replay / keystroke logging — no analytics libraries exist.
- Age gate / COPPA — no accounts; add a "13 or older" line to the waitlist form only.
- Auto-renewal disclosure — no payment code.
- Upload disclaimer — no upload feature.
- Cookie banner — the only cookie is the strictly-necessary `expiresoon_demo`
  session cookie (ePrivacy Art. 5(3) exempts it). Document it in the privacy policy.
