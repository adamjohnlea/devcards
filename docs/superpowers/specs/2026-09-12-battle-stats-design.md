# Battle Stats Design

Date: 2026-09-12
Status: Approved design, pending implementation plan

## Goal

Give every Dev Card six battle stats (0–99) derived from GitHub data, shown on a flippable card back. This is the foundation for a future Top Trumps mode and, later, a deck-builder against an AI. No game mechanics are in scope here.

## Balance principle

Whether fame wins depends on the stat. Two stats reward fame, two reward sustained effort, and two are traits any developer can have. Log scaling keeps famous accounts from maxing everything.

## Stats

| Stat | Kind | Raw input | Scale | Cap |
|---|---|---|---|---|
| IMPACT | fame | total stars across all repos (forks included) | log | 1,000,000 |
| INFLUENCE | fame | followers | log | 500,000 |
| CONTRIBUTIONS | effort | merged PRs authored (GitHub search `total_count`) | log | 5,000 |
| MOMENTUM | effort | public events per day (see below) | log | 50 |
| VETERANCY | anyone | account age in years (from `created_at`) | linear | 18 |
| RANGE | anyone | distinct languages across non-fork repos | linear | 12 |

Formulas, both clamped to 0–99 and rounded to the nearest integer:

- log: `99 × log10(1 + raw) / log10(1 + cap)`
- linear: `99 × raw / cap`

Calibration points (verified against these formulas; used as test fixtures):

| Stat | Raw → score |
|---|---|
| IMPACT | 100 → 33, 10,000 → 66, 1,000,000+ → 99 |
| INFLUENCE | 100 → 35, 1,000 → 52, 322,695 (torvalds, Sept 2026) → 96 |
| CONTRIBUTIONS | 10 → 28, 100 → 54, 632 (dhh) → 75 |
| MOMENTUM | 1/day → 17, 5/day → 45, 50+/day → 99 |
| VETERANCY | 2 yrs → 11, 10 yrs → 55, 18+ yrs → 99 |
| RANGE | 1 → 8, 4 → 33, 12+ → 99 |

Any stat with raw 0 scores 0.

### MOMENTUM window

One page of up to 100 public events is fetched. GitHub only returns events from the last 90 days.

- Fewer than 100 events: the page is the full 90-day window, so `rate = count / 90`.
- Exactly 100 events: `rate = 100 / max(1, days between the oldest event and fetchedAt)`, where days are fractional (`milliseconds / 86,400,000`), not rounded. The 1-day floor stops a short burst from producing an extreme rate.
- The rate is measured against the cache entry's `fetchedAt`, not the render time, so a cached card's MOMENTUM does not drift while the entry is fresh.

### Known quirks (accepted)

- CONTRIBUTIONS includes PRs merged into the user's own repos, because that is what GitHub search returns.
- VETERANCY rises as accounts age, so a card's score slowly increases year to year.

## Architecture

All changes stay in `index.html`.

- `STAT_DEFS`: a single table, one entry per stat: key, display label, how to read the raw input, scale (`log` or `linear`), cap, and a raw-value formatter for the card back (e.g. `45.2K ★`). Rebalancing means editing a cap here.
- `computeBattleStats(raw)`: a pure function taking raw inputs and returning `{ key: score }` for all six stats. It has no DOM or network access, so future game modes call it directly.
- Scores are never stored. They are computed from cached raw inputs at render time, so cap changes apply immediately without refetching.

## Data fetching

`fetchUserData()` fires four requests in parallel:

1. `GET /users/{u}` (existing)
2. repo page 1, then remaining pages as today (existing)
3. `GET /users/{u}/events/public?per_page=100` (new, core rate limit)
4. `GET /search/issues?q=author:{u}+type:pr+is:merged&per_page=1` (new, search rate limit: 10/min anonymous, 30/min with a token). Only `total_count` is read.

A fresh card costs 4 requests (plus 1 per additional 100 repos), up from 2. Cached cards cost 0.

## Cache

The cache entry adds raw inputs alongside the existing fields:

- `mergedPrs`: number
- `events`: `{ count, oldestAt }` (`oldestAt` is an ISO timestamp, or `null` when `count` is 0)

`isValidCacheEntry()` requires the new fields. Entries written before this change fail validation and are refetched. No migration code.

## Error handling

No partial data, because partial data would be cached for an hour.

- Any of the requests failing throws via `apiError()`. The card is not rendered and nothing is cached.
- `apiError()` reads `X-RateLimit-Resource`. When it is `search`, the message says "Search rate limit exceeded" with the reset time, instead of implying the hourly core limit is used up.
- A search response with `incomplete_results: true` throws "GitHub search timed out, try again" rather than caching a possibly low CONTRIBUTIONS count.

## Card back UI

### Structure

`.card` has `overflow: hidden`, which flattens 3D transforms and would break a flip. New structure:

- `.card`: keeps the mouse tilt, reveal animation, and click-to-open-GitHub. Loses `overflow: hidden`.
- `.card-flipper`: rotates 180° on the Y axis when flipped.
- `.card-face.front`: the current card contents, unchanged.
- `.card-face.back`: the battle stats face.

Each face owns its border radius, clipping, and `backface-visibility: hidden`. Each face has its own `.card-foil`, so the pointer glare works on both sides (the `--foil-*` variables set on `.card` inherit into both).

### Flip control

- A real `<button>` labeled "↻ FLIP" in the existing action row (SAVE / SHARE / + GITDEX).
- One `flipCard()` function implements the flip.
- `aria-pressed` reflects the state. The accessible label toggles between "Show battle stats" and "Show card front". The hidden face gets `aria-hidden="true"`.
- Every new card render resets to the front.
- Under `prefers-reduced-motion: reduce`, the flip is instant (no transition).

### Back layout

- Header: "BATTLE STATS" and the `@handle`, in the rarity color.
- Six rows, one per stat in `STAT_DEFS` order: label, bar (width = score %, card accent color), score, and the formatted raw value in small type beneath (e.g. `632 merged PRs`).
- Each row is real text, so screen readers announce e.g. "IMPACT 66, 45.2K stars".
- Footer: the same card number line as the front (`#0002741 · DEV-S01`).

### Export

- SAVE exports the face currently showing.
- Front: `devcard-{u}.png`, unchanged.
- Back: `devcard-{u}-back.png`, drawn by a new `drawCardBackToCanvas()` at the same 3× scale.
- Like the front renderer, `drawCardBackToCanvas()` reads values from the back face's DOM, so markup changes to the back must be mirrored in it.

SHARE is unchanged and posts the card link regardless of which face is showing.

## Out of scope

- Game modes (Top Trumps, deck-builder, AI opponent).
- Language types and type advantages.
- Rarity-based stat bonuses.
- Snapshotting stats into GitDex entries. A future game mode can load stats through the cache or refetch.
- An automated test file.

## Verification

Performed in the browser against the running page during implementation. No test file is added. Automated tests are deferred until game logic moves into a separate `.js` file.

- **Score math:** every calibration point in the Stats section matches exactly. Raw 0 scores 0. Values past the cap clamp to 99.
- **MOMENTUM window:** 40 events → 40/90 per day; 100 events with the oldest 2 days old → 50/day; 100 events with the oldest 10 minutes old → floored to 1 day → score 99; 0 events → 0.
- **Requests:** a fresh card for an account with ≤100 repos makes exactly 4 API requests. A cached render makes 0.
- **Cache:** a pre-change cache entry is refetched. Editing a cap in `STAT_DEFS` changes the displayed score with no network request.
- **Real accounts:** dhh, torvalds, sindresorhus, and a small account produce plausible stats.
- **Errors** (with `fetch` stubbed in the console): a search 403 with `X-RateLimit-Resource: search` names the search limit and reset time; `incomplete_results: true` shows the timeout error and caches nothing; a failed events request shows an error and caches nothing.
- **UI:** the flip button toggles faces; `aria-pressed` and `aria-hidden` update; Enter and Space activate it; a new card render resets to the front; card click still opens GitHub; tilt and foil glare work on the back; the reduced-motion rule is present.
- **Export:** SAVE on the front produces `devcard-{u}.png` as before. SAVE on the back produces `devcard-{u}-back.png`, visually matching the on-screen back.
- **Console:** no errors.
