# Battle Stats Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Give every Dev Card six 0–99 battle stats derived from GitHub data, shown on a flippable card back with per-face PNG export.

**Architecture:** A pure stat engine (`STAT_DEFS`, `scoreStat`, `momentumRate`, `battleRawInputs`, `computeBattleStats`) turns raw inputs into scores. `fetchUserData()` gains two parallel requests (public events, merged-PR search) whose raw results are cached alongside existing data. The card becomes two faces inside a `.card-flipper`, with a FLIP button, and SAVE exports whichever face is showing.

**Tech Stack:** Single-file `index.html` (HTML, CSS, vanilla JS). No build, no dependencies, no test runner. Served locally by Laravel Herd at `http://dev-cards.test`.

**Spec:** `docs/superpowers/specs/2026-09-12-battle-stats-design.md`

## Global Constraints

- All app code stays in `index.html`. No new files besides docs. No dependencies.
- Scores are integers 0–99. log: `99 × log10(1 + raw) / log10(1 + cap)`; linear: `99 × raw / cap`; clamped to 99, rounded to nearest integer; raw 0 → 0.
- Caps: IMPACT 1,000,000 (log) · INFLUENCE 500,000 (log) · CONTRIBUTIONS 5,000 (log) · MOMENTUM 50/day (log) · VETERANCY 18 years (linear) · RANGE 12 languages (linear).
- No partial data is ever cached. Any failed request throws via `apiError()`; nothing renders and nothing is cached.
- Scores are never stored; they are computed from cached raw inputs at render time.
- `flipCard()` is the only user-facing flip path.
- CSP is unchanged: `connect-src https://api.github.com` already covers the events and search endpoints.
- Commit messages end with `Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>`. Do not push.

## How verification works (read before Task 1)

There is no test runner. Each "test" is a JavaScript snippet run in the live page:

1. Make sure Herd is serving the repo: `http://dev-cards.test` loads the app.
2. After every edit to `index.html`, reload the page (`mcp__Claude_Browser__navigate` to `http://dev-cards.test/`), because old functions stay in memory until reload.
3. Run the snippet with `mcp__Claude_Browser__javascript_tool` (`action: "javascript_exec"`). It supports top-level `await` and returns the last expression. Each snippet ends in `JSON.stringify(...)` so the result is readable.
4. Compare the returned JSON against the **Expected** block exactly.

GitHub search allows 10 requests/minute without a token. If a snippet reports "Search rate limit exceeded", wait 60 seconds and rerun.

---

### Task 1: Stat engine

Pure functions only. No network, no DOM.

**Files:**
- Modify: `index.html`: insert a new `// ── Battle stats ──` section directly after `function computeStats(repos) { ... }` (currently ends at line 1076) and before `const VALID_USERNAME = ...` (currently line 1078).

**Interfaces:**
- Consumes: `fmtNum(n)` (existing, defined near the top of the script).
- Produces:
  - `const EVENTS_PAGE_SIZE = 100`, `const EVENTS_WINDOW_DAYS = 30`, `const MS_PER_DAY = 86400000`
  - `const STAT_DEFS`: array of `{ key: string, label: string, scale: 'log'|'linear', cap: number, raw: (rawInputs) => number, format: (value: number) => string }`, in order impact, influence, contributions, momentum, veterancy, range
  - `scoreStat(def, raw: number) => number` (integer 0–99; throws on non-finite raw)
  - `momentumRate(events: {count: number, oldestAt: string|null}, fetchedAt: number) => number` (events per day)
  - `battleRawInputs(data) => { stars, followers, mergedPrs, momentumRate, accountYears, languageCount }` where `data` is a cache entry: `{ stars, langs, mergedPrs, events, fetchedAt, user: { followers, created_at, ... } }`
  - `computeBattleStats(rawInputs) => { impact, influence, contributions, momentum, veterancy, range }` (keys in `STAT_DEFS` order)

- [ ] **Step 1: Write the failing test**

Reload `http://dev-cards.test/`, then run:

```js
const def = k => STAT_DEFS.find(d => d.key === k);
const cases = [
  ['impact', 0, 0], ['impact', 100, 33], ['impact', 10000, 66], ['impact', 1000000, 99], ['impact', 5000000, 99],
  ['influence', 100, 35], ['influence', 1000, 52], ['influence', 322695, 96],
  ['contributions', 10, 28], ['contributions', 100, 54], ['contributions', 632, 75],
  ['momentum', 1, 17], ['momentum', 5, 45], ['momentum', 50, 99], ['momentum', 80, 99],
  ['veterancy', 2, 11], ['veterancy', 10, 55], ['veterancy', 18.6, 99],
  ['range', 0, 0], ['range', 1, 8], ['range', 4, 33], ['range', 12, 99], ['range', 15, 99],
];
const scoreFailures = cases
  .map(([k, raw, want]) => ({ k, raw, want, got: scoreStat(def(k), raw) }))
  .filter(c => c.got !== c.want);

const now = Date.parse('2026-09-12T12:00:00Z');
const ago = ms => new Date(now - ms).toISOString();
const momentumFailures = [
  [{ count: 40,  oldestAt: ago(80 * MS_PER_DAY) }, 40 / 30],
  [{ count: 100, oldestAt: ago(2 * MS_PER_DAY) },  50],
  [{ count: 100, oldestAt: ago(20 * MS_PER_DAY) }, 5],
  [{ count: 100, oldestAt: ago(10 * 60 * 1000) },  100],
  [{ count: 0,   oldestAt: null },                 0],
].map(([events, want]) => ({ events, want, got: momentumRate(events, now) }))
 .filter(c => Math.abs(c.got - c.want) > 1e-9);

const data = {
  stars: 10000, langs: ['Ruby', 'Go', 'JavaScript', 'Shell'], mergedPrs: 632,
  events: { count: 100, oldestAt: ago(20 * MS_PER_DAY) }, fetchedAt: now,
  user: { followers: 1000, created_at: new Date(now - 10 * 365.25 * MS_PER_DAY).toISOString() },
};
const raw = battleRawInputs(data);
const stats = computeBattleStats(raw);
const expectedStats = { impact: 66, influence: 52, contributions: 75, momentum: 45, veterancy: 55, range: 33 };

let nanThrows = false;
try { scoreStat(def('impact'), NaN); } catch (err) { nanThrows = true; }

JSON.stringify({
  scoreFailures, momentumFailures,
  statsOk: JSON.stringify(stats) === JSON.stringify(expectedStats),
  stats,
  formatted: STAT_DEFS.map(d => d.format(d.raw(raw))),
  singular: [def('range').format(1), def('veterancy').format(1.5), def('momentum').format(40 / 90)],
  nanThrows,
})
```

- [ ] **Step 2: Run test to verify it fails**

Expected: the call errors with `ReferenceError: STAT_DEFS is not defined` (or `scoreStat is not defined`).

- [ ] **Step 3: Write minimal implementation**

Insert after the closing `}` of `computeStats`:

```js
// ── Battle stats ──────────────────────────────────────────────────────────────
// Spec: docs/superpowers/specs/2026-09-12-battle-stats-design.md

const EVENTS_PAGE_SIZE = 100;
const EVENTS_WINDOW_DAYS = 30;
const MS_PER_DAY = 86400000;

function formatRate(v) {
  return v < 10 ? v.toFixed(1).replace(/\.0$/, '') : String(Math.round(v));
}

// Rebalance by editing caps. Order here is the order on the card back.
const STAT_DEFS = [
  { key: 'impact',        label: 'IMPACT',        scale: 'log',    cap: 1000000, raw: r => r.stars,         format: v => `${fmtNum(v)} stars` },
  { key: 'influence',     label: 'INFLUENCE',     scale: 'log',    cap: 500000,  raw: r => r.followers,     format: v => `${fmtNum(v)} followers` },
  { key: 'contributions', label: 'CONTRIBUTIONS', scale: 'log',    cap: 5000,    raw: r => r.mergedPrs,     format: v => `${fmtNum(v)} merged PRs` },
  { key: 'momentum',      label: 'MOMENTUM',      scale: 'log',    cap: 50,      raw: r => r.momentumRate,  format: v => `${formatRate(v)} events per day` },
  { key: 'veterancy',     label: 'VETERANCY',     scale: 'linear', cap: 18,      raw: r => r.accountYears,  format: v => { const n = Math.floor(v); return `${n} ${n === 1 ? 'year' : 'years'}`; } },
  { key: 'range',         label: 'RANGE',         scale: 'linear', cap: 12,      raw: r => r.languageCount, format: v => `${v} ${v === 1 ? 'language' : 'languages'}` },
];

function scoreStat(def, raw) {
  if (!Number.isFinite(raw)) throw new Error(`Stat ${def.key} got non-finite raw value: ${raw}`);
  if (raw <= 0) return 0;
  const ratio = def.scale === 'log'
    ? Math.log10(1 + raw) / Math.log10(1 + def.cap)
    : raw / def.cap;
  return Math.round(99 * Math.min(1, ratio));
}

// One page of events: a short page is the whole 30-day window; a full page covers
// only back to its oldest event, floored at 1 day so a burst can't spike the rate.
function momentumRate(events, fetchedAt) {
  if (events.count < EVENTS_PAGE_SIZE) return events.count / EVENTS_WINDOW_DAYS;
  const days = (fetchedAt - Date.parse(events.oldestAt)) / MS_PER_DAY;
  return EVENTS_PAGE_SIZE / Math.max(1, days);
}

function battleRawInputs(data) {
  return {
    stars: data.stars,
    followers: data.user.followers,
    mergedPrs: data.mergedPrs,
    momentumRate: momentumRate(data.events, data.fetchedAt),
    accountYears: (data.fetchedAt - Date.parse(data.user.created_at)) / (365.25 * MS_PER_DAY),
    languageCount: data.langs.length,
  };
}

function computeBattleStats(rawInputs) {
  return Object.fromEntries(STAT_DEFS.map(def => [def.key, scoreStat(def, def.raw(rawInputs))]));
}
```

- [ ] **Step 4: Run test to verify it passes**

Reload, rerun the Step 1 snippet.

Expected:

```json
{"scoreFailures":[],"momentumFailures":[],"statsOk":true,"stats":{"impact":66,"influence":52,"contributions":75,"momentum":45,"veterancy":55,"range":33},"formatted":["10K stars","1K followers","632 merged PRs","5 events per day","10 years","4 languages"],"singular":["1 language","1 year","0.4 events per day"],"nanThrows":true}
```

Also run `JSON.stringify(await (async () => { await renderCard('torvalds'); return document.getElementById('statusMsg').textContent; })())` and confirm the existing card still renders (expected `""`, or a rate-limit message if you are rate limited; a `ReferenceError` means the insertion broke the script).

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Add battle stat engine

STAT_DEFS table plus pure scoreStat, momentumRate, battleRawInputs, and
computeBattleStats. Not yet wired to data or UI.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 2: Fetch events and merged PRs, cache raw inputs

**Files:**
- Modify: `index.html`
  - `apiError(res)` (currently lines 920–940): name the search limit
  - `fetchUserData(username, onProgress)` (currently lines 953–988): two new parallel requests, return `events` and `mergedPrs`
  - `isValidCacheEntry(e)` (currently lines 999–1008): require new fields
  - `loadCardData(username, onProgress)` (currently lines 1047–1067): store new fields
- Modify: `CLAUDE.md`: step 2 of the card generation flow
- Modify: `README.md`: the "On card generation, the app fetches" list

**Interfaces:**
- Consumes: `EVENTS_PAGE_SIZE`, `battleRawInputs`, `computeBattleStats` (Task 1).
- Produces:
  - `fetchUserData(username, onProgress) => Promise<{ user, repos, events: { count: number, oldestAt: string|null }, mergedPrs: number }>`
  - Cache entry shape (returned as `data` by `loadCardData`): `{ user: { login, name, avatar_url, id, followers, public_repos, created_at, hireable, blog }, stars, langs, mergedPrs, events: { count, oldestAt }, fetchedAt }`

- [ ] **Step 1: Write the failing tests**

Reload `http://dev-cards.test/`, then run **test A (real API, uses 4 requests including 1 search)**:

```js
const api = () => performance.getEntriesByType('resource')
  .filter(e => e.name.startsWith('https://api.github.com'))
  .map(e => e.name.replace('https://api.github.com', ''));
localStorage.removeItem('devcard_cache:octocat');
performance.clearResourceTimings();
await renderCard('octocat');
const first = api();
const entry = JSON.parse(localStorage.getItem('devcard_cache:octocat'));
performance.clearResourceTimings();
await renderCard('octocat');
const second = api();

const { mergedPrs, events, ...legacyShape } = entry;
localStorage.setItem('devcard_cache:legacy', JSON.stringify(legacyShape));
const legacyRejected = readCache('legacy') === null && localStorage.getItem('devcard_cache:legacy') === null;

JSON.stringify({
  firstCount: first.length,
  hasEvents: first.some(u => u.startsWith('/users/octocat/events/public?per_page=100')),
  hasSearch: first.some(u => u.startsWith('/search/issues?q=author%3Aoctocat%20type%3Apr%20is%3Amerged')),
  secondCount: second.length,
  mergedPrs: entry.mergedPrs,
  events: entry.events,
  stats: computeBattleStats(battleRawInputs(entry)),
  legacyRejected,
  status: document.getElementById('statusMsg').textContent,
})
```

Then run **test B (stubbed fetch, no real requests)**:

```js
const realFetch = window.fetch;
const json = (body, status = 200, headers = {}) =>
  new Response(JSON.stringify(body), { status, headers: { 'Content-Type': 'application/json', ...headers } });
const fakeUser = {
  login: 'stubuser', name: 'Stub', avatar_url: 'https://avatars.githubusercontent.com/u/1?v=4', id: 1,
  followers: 5, public_repos: 1, created_at: '2020-01-01T00:00:00Z', hireable: null, blog: '',
};
function install(searchResponse, eventsResponse) {
  window.fetch = async url => {
    if (url.includes('/search/issues')) return searchResponse();
    if (url.includes('/events/public')) return eventsResponse();
    if (url.includes('/repos')) return json([]);
    if (url.includes('/users/stubuser')) return json(fakeUser);
    throw new Error('Unexpected fetch ' + url);
  };
}
const status = () => document.getElementById('statusMsg').textContent;
const cached = () => localStorage.getItem('devcard_cache:stubuser');
const reset = String(Math.floor(Date.now() / 1000) + 60);
const okSearch = () => json({ total_count: 3, incomplete_results: false, items: [] });
const results = {};
localStorage.removeItem('devcard_cache:stubuser');

install(() => json({ message: 'API rate limit exceeded' }, 403,
  { 'X-RateLimit-Resource': 'search', 'X-RateLimit-Remaining': '0', 'X-RateLimit-Reset': reset }), () => json([]));
await renderCard('stubuser');
results.searchLimit = { status: status(), cached: cached() !== null };

install(() => json({ total_count: 3, incomplete_results: true, items: [] }), () => json([]));
await renderCard('stubuser');
results.incomplete = { status: status(), cached: cached() !== null };

install(okSearch, () => json({ message: 'Server Error' }, 500));
await renderCard('stubuser');
results.eventsFail = { status: status(), cached: cached() !== null };

install(okSearch, () => json([{ created_at: '2026-09-10T00:00:00Z' }, { created_at: '2026-09-01T00:00:00Z' }]));
await renderCard('stubuser');
const stored = JSON.parse(cached());
results.success = { status: status(), mergedPrs: stored.mergedPrs, events: stored.events };

window.fetch = realFetch;
localStorage.removeItem('devcard_cache:stubuser');
JSON.stringify(results)
```

- [ ] **Step 2: Run tests to verify they fail**

Expected, test A: the snippet throws `TypeError: Cannot read properties of undefined (reading 'count')` from `momentumRate`, because the cache entry has no `events` field yet. (Only 2 API requests were made; the events and search requests don't exist yet.)

Expected, test B: `searchLimit.status` is `""` (the stubbed search is never requested), and `success.mergedPrs` is `undefined`.

- [ ] **Step 3: Implement `apiError` search-limit message**

In `apiError(res)`, replace the 403/429 branch:

```js
  if (res.status === 403 || res.status === 429) {
    const resetsAt = reset ? ` Resets at ${new Date(reset * 1000).toLocaleTimeString()}.` : '';
    const tokenHint = getToken() ? '' : ' Add a GitHub token (⚙) for 5,000 req/hr.';
    return new Error(`Rate limit exceeded.${resetsAt}${tokenHint}`);
  }
```

with:

```js
  if (res.status === 403 || res.status === 429) {
    const isSearch  = res.headers.get('X-RateLimit-Resource') === 'search';
    const resetsAt  = reset ? ` Resets at ${new Date(reset * 1000).toLocaleTimeString()}.` : '';
    const tokenHint = getToken() ? ''
      : isSearch ? ' Add a GitHub token (⚙) for 30 searches/min.'
      : ' Add a GitHub token (⚙) for 5,000 req/hr.';
    return new Error(`${isSearch ? 'Search rate limit' : 'Rate limit'} exceeded.${resetsAt}${tokenHint}`);
  }
```

- [ ] **Step 4: Implement the new requests in `fetchUserData`**

Replace the opening of `fetchUserData` through `const repos = [...lastPage];`:

```js
async function fetchUserData(username, onProgress) {
  const enc = encodeURIComponent(username);
  const opts = { headers: githubHeaders() };
  const [userRes, firstPageRes] = await Promise.all([
    fetch(`https://api.github.com/users/${enc}`, opts),
    fetch(repoPageUrl(enc, 1), opts)
  ]);

  if (!userRes.ok) throw await apiError(userRes);
  if (!firstPageRes.ok) throw await apiError(firstPageRes);

  const user = await userRes.json();
  let lastPage = await firstPageRes.json();
  const repos = [...lastPage];
```

with:

```js
async function fetchUserData(username, onProgress) {
  const enc = encodeURIComponent(username);
  const opts = { headers: githubHeaders() };
  const mergedPrQuery = encodeURIComponent(`author:${username} type:pr is:merged`);
  const [userRes, firstPageRes, eventsRes, searchRes] = await Promise.all([
    fetch(`https://api.github.com/users/${enc}`, opts),
    fetch(repoPageUrl(enc, 1), opts),
    fetch(`https://api.github.com/users/${enc}/events/public?per_page=${EVENTS_PAGE_SIZE}`, opts),
    fetch(`https://api.github.com/search/issues?q=${mergedPrQuery}&per_page=1`, opts)
  ]);

  // User first, so a missing user reports "User not found" rather than a search error.
  if (!userRes.ok) throw await apiError(userRes);
  if (!firstPageRes.ok) throw await apiError(firstPageRes);
  if (!eventsRes.ok) throw await apiError(eventsRes);
  if (!searchRes.ok) throw await apiError(searchRes);

  const user = await userRes.json();
  const eventList = await eventsRes.json();
  const search = await searchRes.json();
  // A timed-out search can under-count; caching it would pin a wrong CONTRIBUTIONS score for an hour.
  if (search.incomplete_results) throw new Error('GitHub search timed out, try again');

  let lastPage = await firstPageRes.json();
  const repos = [...lastPage];
```

Then replace the function's final `return { user, repos };` with:

```js
  // Events arrive newest first, so the last one is the oldest.
  const events = {
    count: eventList.length,
    oldestAt: eventList.length ? eventList[eventList.length - 1].created_at : null,
  };
  return { user, repos, events, mergedPrs: search.total_count };
```

- [ ] **Step 5: Implement cache validation and storage**

In `isValidCacheEntry(e)`, replace:

```js
    Number.isFinite(u.id) && Number.isFinite(u.followers) && Number.isFinite(u.public_repos) &&
    typeof u.created_at === 'string';
```

with:

```js
    Number.isFinite(u.id) && Number.isFinite(u.followers) && Number.isFinite(u.public_repos) &&
    typeof u.created_at === 'string' &&
    Number.isFinite(e.mergedPrs) &&
    !!e.events && Number.isInteger(e.events.count) && e.events.count >= 0 &&
    (e.events.count === 0
      ? e.events.oldestAt === null
      : typeof e.events.oldestAt === 'string' && Number.isFinite(Date.parse(e.events.oldestAt)));
```

In `loadCardData`, replace:

```js
  const { user, repos } = await fetchUserData(username, onProgress);
  const { stars, langs } = computeStats(repos);
  const { login, name, avatar_url, id, followers, public_repos, created_at, hireable, blog } = user;
  const data = {
    user: { login, name, avatar_url, id, followers, public_repos, created_at, hireable, blog },
    stars, langs, fetchedAt: Date.now(),
  };
```

with:

```js
  const { user, repos, events, mergedPrs } = await fetchUserData(username, onProgress);
  const { stars, langs } = computeStats(repos);
  const { login, name, avatar_url, id, followers, public_repos, created_at, hireable, blog } = user;
  // Raw inputs only: battle scores are computed at render so cap changes apply without refetching.
  const data = {
    user: { login, name, avatar_url, id, followers, public_repos, created_at, hireable, blog },
    stars, langs, mergedPrs, events, fetchedAt: Date.now(),
  };
```

- [ ] **Step 6: Run tests to verify they pass**

Reload, rerun test A. Expected (`mergedPrs`, `events`, and `stats` values vary with octocat's live data; check types and ranges):

```json
{"firstCount":4,"hasEvents":true,"hasSearch":true,"secondCount":0,"mergedPrs":<integer ≥ 0>,"events":{"count":<0–100>,"oldestAt":<ISO string, or null if count is 0>},"stats":{"impact":<0–99>,"influence":<0–99>,"contributions":<0–99>,"momentum":<0–99>,"veterancy":<0–99>,"range":<0–99>},"legacyRejected":true,"status":""}
```

Reload, rerun test B. Expected (`Resets at` time varies; the token hint is absent if a token is saved):

```json
{"searchLimit":{"status":"Search rate limit exceeded. Resets at <time>. Add a GitHub token (⚙) for 30 searches/min.","cached":false},"incomplete":{"status":"GitHub search timed out, try again","cached":false},"eventsFail":{"status":"Server Error","cached":false},"success":{"status":"","mergedPrs":3,"events":{"count":2,"oldestAt":"2026-09-01T00:00:00Z"}}}
```

- [ ] **Step 7: Update docs**

In `CLAUDE.md`, replace the whole of step 2 in the "Card generation flow" list (it begins "2. `loadCardData()` returns a fresh") with:

```markdown
2. `loadCardData()` returns a fresh `localStorage` cache entry if one exists; otherwise `fetchUserData()` fires four requests in parallel: `GET /users/{u}`, repo page 1, `GET /users/{u}/events/public?per_page=100` (MOMENTUM), and `GET /search/issues?q=author:{u} type:pr is:merged&per_page=1` (CONTRIBUTIONS; search has its own limit of 10/min, 30/min with a token). Remaining repo pages follow in parallel, then it keeps fetching while a page comes back full (guards against a stale `public_repos`). Repos use `sort=full_name` because `sort=updated` can shift repos across page boundaries mid-fetch. `githubHeaders()` adds a Bearer token if one is saved. Any failed request throws via `apiError()` (404 → "User not found", 403/429 → a message naming the core or search limit, with reset time and a token hint), and a search with `incomplete_results: true` throws too. Never substitute partial data, since it would be cached for an hour. Cache entries hold raw inputs (`stars`, `langs`, `mergedPrs`, `events: {count, oldestAt}`, profile fields), never battle scores.
```

In `README.md`, replace:

```markdown
On card generation, the app fetches:
- `GET /users/{username}`: profile data
- `GET /users/{username}/repos?per_page=100&sort=full_name&page=N`: every page of repositories, so star totals are accurate for accounts with more than 100 repos (a 1,000-repo account costs about 11 requests)
```

with:

```markdown
On card generation, the app fetches:
- `GET /users/{username}`: profile data
- `GET /users/{username}/repos?per_page=100&sort=full_name&page=N`: every page of repositories, so star totals are accurate for accounts with more than 100 repos
- `GET /users/{username}/events/public?per_page=100`: recent public activity, for the MOMENTUM stat
- `GET /search/issues?q=author:{username} type:pr is:merged&per_page=1`: merged pull request count, for the CONTRIBUTIONS stat

A fresh card costs 4 requests for accounts with up to 100 repos, plus 1 per additional 100 repos. GitHub search has its own rate limit: 10 requests/minute anonymous, 30 with a token.
```

- [ ] **Step 8: Commit**

```bash
git add index.html CLAUDE.md README.md
git commit -m "Fetch public events and merged PR count for battle stats

Two more parallel requests per fresh card. Raw results are cached;
failures, search rate limits, and incomplete search results surface as
errors and are never cached. Pre-change cache entries fail validation
and are refetched.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 3: Card back and flip

**Files:**
- Modify: `index.html`
  - CSS `.card` (currently lines 132–143): remove `overflow: hidden`
  - CSS reduced-motion block (currently lines 197–199): add the flipper
  - CSS: add face, flipper, and back styles after `.card-number { ... }` (currently ends line 394)
  - HTML card markup (currently lines 793–839): wrap in faces, add back face
  - HTML `.card-actions` (currently lines 841–845): add FLIP button
  - JS `renderCard` (currently lines 1086–1207): shared background, render back, reset flip
  - JS `downloadCard` (currently lines 1220–1238): shorter saving label
  - JS: add `renderBattleStats`, `setCardFlipped`, `isCardFlipped`, `flipCard` after `quickGen` (currently lines 1209–1212)
- Modify: `CLAUDE.md`, `README.md`

**Interfaces:**
- Consumes: `STAT_DEFS`, `battleRawInputs`, `computeBattleStats` (Task 1); cache entry shape with `mergedPrs` and `events` (Task 2).
- Produces:
  - DOM ids: `cardFront`, `cardBack`, `cardBackBg`, `backTitle`, `backHandle`, `battleStats`, `backCardNumber`, `flipBtn`
  - Each stat row: `li.battle-stat[data-stat][data-score]` containing `.battle-stat-label`, `.battle-stat-bar > .battle-stat-fill`, `.battle-stat-score`, `.battle-stat-raw`
  - `renderBattleStats(data, rarity, accentColor) => void`
  - `setCardFlipped(flipped: boolean) => void`, `isCardFlipped() => boolean`, `flipCard() => void`

- [ ] **Step 1: Write the failing test**

Reload `http://dev-cards.test/`, then run (octocat is cached from Task 2 if it was within the hour; otherwise this costs 4 requests):

```js
await renderCard('octocat');
const entry = readCache('octocat');
const expected = computeBattleStats(battleRawInputs(entry));
const card = document.getElementById('devCard');
const btn = document.getElementById('flipBtn');
const rows = [...document.querySelectorAll('#battleStats .battle-stat')].map(li => ({
  key: li.dataset.stat,
  score: Number(li.dataset.score),
  label: li.querySelector('.battle-stat-label').textContent,
  shown: li.querySelector('.battle-stat-score').textContent,
  raw: li.querySelector('.battle-stat-raw').textContent,
  fill: li.querySelector('.battle-stat-fill').style.width,
}));
const state = () => ({
  flipped: card.classList.contains('is-flipped'),
  label: btn.getAttribute('aria-label'),
  pressed: btn.getAttribute('aria-pressed'),
  frontHidden: document.getElementById('cardFront').getAttribute('aria-hidden'),
  backHidden: document.getElementById('cardBack').getAttribute('aria-hidden'),
});
const before = state();
flipCard();
const after = state();
await renderCard('octocat');
const afterRerender = state();
const rules = [...document.styleSheets].flatMap(s => { try { return [...s.cssRules]; } catch (err) { return []; } });
JSON.stringify({
  keys: rows.map(r => r.key),
  labels: rows.map(r => r.label),
  rowsMatch: rows.length === 6 && rows.every(r =>
    r.score === expected[r.key] && r.shown === String(expected[r.key]) && r.fill === `${expected[r.key]}%`),
  raws: rows.map(r => r.raw),
  before, after, afterRerender,
  reducedMotionRule: rules.some(r => r.media && r.media.mediaText.includes('prefers-reduced-motion') && r.cssText.includes('.card-flipper')),
  cardOverflow: getComputedStyle(card).overflow,
  cardClickOpensGithub: typeof card.onclick === 'function',
  flipBtnOutsideCard: !card.contains(btn),
  actionsFit: document.querySelector('.card-actions').scrollWidth <= 340,
})
```

- [ ] **Step 2: Run test to verify it fails**

Expected: `TypeError: Cannot read properties of null (reading 'getAttribute')` (no `#flipBtn` yet).

- [ ] **Step 3: Update card CSS**

In `.card`, delete the line `    overflow: hidden;`. Leave every other property.

Replace the reduced-motion block:

```css
  @media (prefers-reduced-motion: reduce) {
    .card-foil { animation: none; }
  }
```

with:

```css
  @media (prefers-reduced-motion: reduce) {
    .card-foil { animation: none; }
    .card-flipper { transition: none; }
  }
```

After the closing `}` of `.card-number`, add:

```css
  /* ── CARD FACES ───────────────────────── */
  /* .card must not regain overflow: hidden; it flattens 3D and breaks the flip. */
  .card-flipper {
    position: relative;
    transform-style: preserve-3d;
    transition: transform 0.7s cubic-bezier(0.23, 1, 0.32, 1);
  }

  .card.is-flipped .card-flipper { transform: rotateY(180deg); }

  .card-face {
    border-radius: 12px;
    overflow: hidden;
    backface-visibility: hidden;
    -webkit-backface-visibility: hidden;
  }

  /* The front stays in flow and sets the card's height; the back overlays it. */
  .card-front {
    position: relative;
    min-height: 520px;
  }

  .card-back {
    position: absolute;
    inset: 0;
    transform: rotateY(180deg);
  }

  .card-back-handle {
    font-family: 'Space Mono', monospace;
    font-size: 11px;
    color: rgba(255,255,255,0.45);
    letter-spacing: 0.15em;
    line-height: 1;
  }

  .battle-stats {
    list-style: none;
    margin: 0;
    padding: 8px 18px 0;
    display: flex;
    flex-direction: column;
    gap: 18px;
  }

  .battle-stat-row {
    display: grid;
    grid-template-columns: 112px 1fr 28px;
    align-items: center;
    gap: 10px;
  }

  .battle-stat-label {
    font-family: 'Bebas Neue', sans-serif;
    font-size: 15px;
    letter-spacing: 0.12em;
    color: #fff;
    line-height: 1;
  }

  .battle-stat-bar {
    height: 6px;
    background: rgba(255,255,255,0.08);
    border-radius: 3px;
    overflow: hidden;
  }

  .battle-stat-fill {
    height: 100%;
    border-radius: 3px;
  }

  .battle-stat-score {
    font-family: 'Bebas Neue', sans-serif;
    font-size: 20px;
    color: #fff;
    text-align: right;
    line-height: 1;
  }

  .battle-stat-raw {
    font-family: 'Space Mono', monospace;
    font-size: 9px;
    letter-spacing: 0.1em;
    color: rgba(255,255,255,0.4);
    margin-top: 4px;
    line-height: 1;
  }
```

- [ ] **Step 4: Update card markup and action row**

Replace the card element, from `<div class="card" id="devCard">` through its closing `</div>` (the one just before `</div>` closing `.card-scene`), with the following. The front face's inner markup is the existing markup, unchanged:

```html
    <div class="card" id="devCard">
      <div class="card-flipper">
        <div class="card-face card-front" id="cardFront" aria-hidden="false">
          <div class="card-bg" id="cardBg"></div>
          <div class="card-foil"></div>
          <div class="card-inner">
            <div class="card-header">
              <div class="card-series">DEV CARDS // S01</div>
              <div class="rarity-block">
                <div class="rarity-badge" id="rarityBadge"></div>
                <div class="rarity-modifier hidden" id="rarityModifier"></div>
              </div>
            </div>
            <div class="card-portrait-wrap">
              <div class="card-portrait-frame">
                <img class="card-avatar" id="cardAvatar" src="" alt="avatar" crossorigin="anonymous" />
              </div>
            </div>
            <div class="card-name-block">
              <div class="card-name" id="cardName"></div>
              <div class="card-handle" id="cardHandle"></div>
            </div>
            <div class="card-ability" id="cardAbility">
              <div class="card-ability-label">SPECIAL ABILITY</div>
              <div class="card-ability-text" id="abilityText"></div>
            </div>
            <div class="card-stats">
              <div class="stat-cell">
                <div class="stat-val" id="statRepos">—</div>
                <div class="stat-label">Repos</div>
              </div>
              <div class="stat-cell">
                <div class="stat-val" id="statStars">—</div>
                <div class="stat-label">Stars</div>
              </div>
              <div class="stat-cell">
                <div class="stat-val" id="statFollowers">—</div>
                <div class="stat-label">Followers</div>
              </div>
            </div>
            <div class="card-footer">
              <div class="card-lang-badges" id="langBadges"></div>
              <div class="card-meta">
                <div class="card-vintage hidden" id="cardVintage"></div>
                <div class="card-number" id="cardNumber"></div>
              </div>
            </div>
          </div>
        </div>
        <div class="card-face card-back" id="cardBack" aria-hidden="true">
          <div class="card-bg" id="cardBackBg"></div>
          <div class="card-foil"></div>
          <div class="card-inner">
            <div class="card-header">
              <div class="card-series" id="backTitle">BATTLE STATS</div>
              <div class="card-back-handle" id="backHandle"></div>
            </div>
            <ul class="battle-stats" id="battleStats"></ul>
            <div class="card-footer">
              <div></div>
              <div class="card-meta">
                <div class="card-number" id="backCardNumber"></div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
```

Replace the `.card-actions` block with:

```html
  <div class="card-actions">
    <button class="action-btn" id="flipBtn" onclick="flipCard()" aria-label="Flip to battle stats">↻ FLIP</button>
    <button class="action-btn" id="downloadBtn" onclick="downloadCard()">⬇ SAVE</button>
    <button class="action-btn share-btn" onclick="shareCard()">𝕏 SHARE</button>
    <button class="action-btn gitdex-btn" id="gitdexBtn" onclick="addToGitDex()">+ GITDEX</button>
  </div>
```

- [ ] **Step 5: Add back-face and flip functions**

After the closing `}` of `quickGen`, add:

```js
// ── Card back ─────────────────────────────────────────────────────────────────

function renderBattleStats(data, rarity, accentColor) {
  const raw = battleRawInputs(data);
  const scores = computeBattleStats(raw);

  document.getElementById('backTitle').style.color = rarity.color;
  document.getElementById('backHandle').textContent = `@${data.user.login}`;
  document.getElementById('backCardNumber').textContent = `#${String(data.user.id).padStart(7,'0')} · DEV-S01`;

  const list = document.getElementById('battleStats');
  list.innerHTML = '';
  STAT_DEFS.forEach(def => {
    const score = scores[def.key];

    const li = document.createElement('li');
    li.className = 'battle-stat';
    li.dataset.stat = def.key;
    li.dataset.score = String(score);

    const row = document.createElement('div');
    row.className = 'battle-stat-row';

    const label = document.createElement('span');
    label.className = 'battle-stat-label';
    label.textContent = def.label;

    const bar = document.createElement('div');
    bar.className = 'battle-stat-bar';
    bar.setAttribute('aria-hidden', 'true');
    const fill = document.createElement('div');
    fill.className = 'battle-stat-fill';
    fill.style.width = `${score}%`;
    fill.style.background = accentColor;
    bar.appendChild(fill);

    const scoreEl = document.createElement('span');
    scoreEl.className = 'battle-stat-score';
    scoreEl.textContent = String(score);

    row.append(label, bar, scoreEl);

    const rawEl = document.createElement('div');
    rawEl.className = 'battle-stat-raw';
    rawEl.textContent = def.format(def.raw(raw));

    li.append(row, rawEl);
    list.appendChild(li);
  });
}

function isCardFlipped() {
  return document.getElementById('devCard').classList.contains('is-flipped');
}

function setCardFlipped(flipped) {
  document.getElementById('devCard').classList.toggle('is-flipped', flipped);
  document.getElementById('cardFront').setAttribute('aria-hidden', String(flipped));
  document.getElementById('cardBack').setAttribute('aria-hidden', String(!flipped));
  document.getElementById('flipBtn').setAttribute('aria-label', flipped ? 'Flip to card front' : 'Flip to battle stats');
}

function flipCard() {
  if (!currentCardData) return;
  setCardFlipped(!isCardFlipped());
}
```

- [ ] **Step 6: Wire into `renderCard` and shorten the saving label**

In `renderCard`, replace:

```js
    // Card BG
    const cardBg = document.getElementById('cardBg');
    cardBg.style.background = `
      radial-gradient(ellipse at 50% 0%, ${accentColor}22 0%, transparent 65%),
      linear-gradient(170deg, ${rarity.grad[0]} 0%, ${rarity.grad[1]} 100%)
    `;
```

with:

```js
    // Card BG (both faces)
    const cardBgStyle = `
      radial-gradient(ellipse at 50% 0%, ${accentColor}22 0%, transparent 65%),
      linear-gradient(170deg, ${rarity.grad[0]} 0%, ${rarity.grad[1]} 100%)
    `;
    document.getElementById('cardBg').style.background = cardBgStyle;
    document.getElementById('cardBackBg').style.background = cardBgStyle;
```

In `renderCard`, replace:

```js
    status.textContent = cacheError ? `Card not cached: ${cacheError.message}` : '';
```

with:

```js
    renderBattleStats(data, rarity, accentColor);
    setCardFlipped(false);

    status.textContent = cacheError ? `Card not cached: ${cacheError.message}` : '';
```

In `downloadCard`, replace `btn.textContent = '⏳ SAVING...';` with `btn.textContent = '⏳ SAVING';` (with four buttons, the longer label overflows the 340px action row).

- [ ] **Step 7: Run test to verify it passes**

Reload, rerun the Step 1 snippet. Expected (`raws` values vary with octocat's live data):

```json
{"keys":["impact","influence","contributions","momentum","veterancy","range"],"labels":["IMPACT","INFLUENCE","CONTRIBUTIONS","MOMENTUM","VETERANCY","RANGE"],"rowsMatch":true,"raws":["<n> stars","<n> followers","<n> merged PRs","<n> events per day","<n> years","<n> languages"],"before":{"flipped":false,"label":"Flip to battle stats","pressed":null,"frontHidden":"false","backHidden":"true"},"after":{"flipped":true,"label":"Flip to card front","pressed":null,"frontHidden":"true","backHidden":"false"},"afterRerender":{"flipped":false,"label":"Flip to battle stats","pressed":null,"frontHidden":"false","backHidden":"true"},"reducedMotionRule":true,"cardOverflow":"visible","cardClickOpensGithub":true,"flipBtnOutsideCard":true,"actionsFit":true}
```

- [ ] **Step 8: Verify keyboard, visuals, and foil**

1. Keyboard: run `document.getElementById('flipBtn').focus(); 'focused'`, then send key `Enter` with `mcp__Claude_Browser__computer` (`action: "key"`), then run `JSON.stringify(isCardFlipped())`. Expected: `true`. Send key `space`, rerun. Expected: `false`.
2. Visuals: take a screenshot of the front. Run `flipCard(); 'flipped'`, wait 1 second, take a screenshot. Expected: the back shows "BATTLE STATS" in the rarity color, the handle at top right, six labeled bars with scores and raw values beneath, and the card number bottom right. Text reads left to right, not mirrored, and no front content bleeds through.
3. Foil on the back: with the card flipped, hover the card with `mcp__Claude_Browser__computer` (`action: "hover"` at the card's center), then run `JSON.stringify(getComputedStyle(document.querySelector('#cardBack .card-foil')).opacity)`. Expected: `"1"`.
4. Cap edits apply without refetching: reload, then run:

   ```js
   await renderCard('octocat');
   const before = document.querySelector('[data-stat="impact"]').dataset.score;
   STAT_DEFS.find(d => d.key === 'impact').cap = 1000;
   performance.clearResourceTimings();
   await renderCard('octocat');
   const after = document.querySelector('[data-stat="impact"]').dataset.score;
   const requests = performance.getEntriesByType('resource').filter(e => e.name.startsWith('https://api.github.com')).length;
   JSON.stringify({ before, after, requests })
   ```

   Expected: `after` is higher than `before` (or both are `"99"` if octocat already maxes IMPACT), and `requests` is `0`. Reload afterwards to restore the real cap.
5. Real accounts: reload, then run the snippet below. It costs roughly 25 requests (sindresorhus alone is about 14) and 4 searches, so save a GitHub token first (⚙) or expect to wait out rate limits.

   ```js
   const out = {};
   for (const u of ['torvalds', 'sindresorhus', 'dhh', 'adamjohnlea']) {
     await renderCard(u);
     const entry = readCache(u);
     out[u] = entry ? computeBattleStats(battleRawInputs(entry)) : document.getElementById('statusMsg').textContent;
   }
   JSON.stringify(out)
   ```

   Expected: every account returns an object (not an error string) with six integers from 0 to 99. torvalds INFLUENCE is about 96, sindresorhus IMPACT is 99, dhh CONTRIBUTIONS is about 75 (±2 as live numbers drift), and adamjohnlea's IMPACT and INFLUENCE are lower than all three others'.
6. Console: `mcp__Claude_Browser__read_console_messages` with `onlyErrors: true`. Expected: no errors from this task.

- [ ] **Step 9: Update docs**

In `CLAUDE.md`, add this paragraph directly after the "Card generation flow" list:

```markdown
**Battle stats** (spec: `docs/superpowers/specs/2026-09-12-battle-stats-design.md`): `STAT_DEFS` is the single table of the six stats (IMPACT, INFLUENCE, CONTRIBUTIONS, MOMENTUM, VETERANCY, RANGE) with scale (`log`/`linear`), cap, raw-input reader, and formatter. Rebalance by editing caps there. `battleRawInputs(cacheEntry)` derives raw inputs (MOMENTUM and VETERANCY are measured against the entry's `fetchedAt`), and `computeBattleStats(raw)` returns `{key: 0–99}`. It is pure with no DOM access, so game modes should call it directly. Scores are computed at render, never cached. `renderBattleStats()` fills the card back.
```

In `CLAUDE.md`, append to the end of the **Styling** paragraph:

```markdown
The card has two faces (`#cardFront`, `#cardBack`) inside `.card-flipper`. Do not put `overflow: hidden` back on `.card`: it flattens 3D and breaks the flip. `flipCard()` is the only user-facing flip path, and `renderCard()` resets to the front with `setCardFlipped(false)`.
```

In `README.md`, add this bullet directly after the **Rarity system** bullet in Features:

```markdown
- **Battle stats** — flip a card (↻ FLIP) to see six 0–99 stats: IMPACT (stars), INFLUENCE (followers), CONTRIBUTIONS (merged PRs), MOMENTUM (recent activity), VETERANCY (account age), and RANGE (languages)
```

- [ ] **Step 10: Commit**

```bash
git add index.html CLAUDE.md README.md
git commit -m "Add flippable card back with battle stats

Card is now two faces inside a 3D flipper; the FLIP button is the only
flip path, resets on each render, and is instant under reduced motion.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 4: Export the showing face

**Files:**
- Modify: `index.html`
  - `drawCardToCanvas()` (currently lines 1433–1602, shifted by earlier tasks): extract shared background into `drawCardBase`
  - Add `drawCardBackToCanvas()` directly after `drawCardToCanvas()`
  - `downloadCard()`: choose renderer and filename by face
- Modify: `CLAUDE.md` (canvas export paragraph), `README.md` (Download bullet)

**Interfaces:**
- Consumes: `isCardFlipped()`, back-face DOM ids and row classes (Task 3); `rrPath`, `hexToRgb` (existing); `currentCardData.rarity`, `currentCardData.accentColor`, `currentCardData.username` (existing).
- Produces:
  - `drawCardBase(ctx, W, H, rarity, accentColor) => void` (clips to the rounded card and paints background gradient plus accent glow)
  - `drawCardBackToCanvas() => HTMLCanvasElement` (3× scale)

- [ ] **Step 1: Capture the front export fingerprint before changing anything**

Reload `http://dev-cards.test/`, then run:

```js
await document.fonts.ready;
await renderCard('octocat');
const c = await drawCardToCanvas();
const digest = await crypto.subtle.digest('SHA-256', new TextEncoder().encode(c.toDataURL('image/png')));
const hash = [...new Uint8Array(digest)].map(b => b.toString(16).padStart(2, '0')).join('');
localStorage.setItem('devcard_test_front_hash', hash);
JSON.stringify({ saved: hash.length === 64 })
```

Expected: `{"saved":true}`

- [ ] **Step 2: Write the failing test**

Run:

```js
await document.fonts.ready;
await renderCard('octocat');
const names = [];
const realClick = HTMLAnchorElement.prototype.click;
HTMLAnchorElement.prototype.click = function () { names.push(this.download); };
setCardFlipped(false);
await downloadCard();
setCardFlipped(true);
await downloadCard();
HTMLAnchorElement.prototype.click = realClick;
setCardFlipped(false);
const back = drawCardBackToCanvas();
JSON.stringify({ names, width: back.width, height: back.height, expectedHeight: document.getElementById('cardBack').offsetHeight * 3 })
```

- [ ] **Step 3: Run test to verify it fails**

Expected: `ReferenceError: drawCardBackToCanvas is not defined`. (If `names` is inspected before the error, it holds two copies of `devcard-octocat.png`.)

- [ ] **Step 4: Extract `drawCardBase` from the front renderer**

In `drawCardToCanvas()`, replace:

```js
  // Clip to card rounded rect
  rrPath(ctx, 0, 0, W, H, 12); ctx.clip();

  // Background gradient
  const bg = ctx.createLinearGradient(0, 0, 0, H);
  bg.addColorStop(0, rarity.grad[0]); bg.addColorStop(1, rarity.grad[1]);
  ctx.fillStyle = bg; ctx.fillRect(0, 0, W, H);

  // Accent radial glow
  const rgb = hexToRgb(accentColor);
  if (rgb) {
    const rg = ctx.createRadialGradient(W/2, 0, 0, W/2, 0, W * 0.65);
    rg.addColorStop(0, `rgba(${rgb},0.13)`); rg.addColorStop(1, 'rgba(0,0,0,0)');
    ctx.fillStyle = rg; ctx.fillRect(0, 0, W, H);
  }
```

with:

```js
  drawCardBase(ctx, W, H, rarity, accentColor);
```

Add this function directly before `async function drawCardToCanvas()`:

```js
// Shared by both face renderers: clip to the rounded card, then background gradient and accent glow.
function drawCardBase(ctx, W, H, rarity, accentColor) {
  rrPath(ctx, 0, 0, W, H, 12); ctx.clip();

  const bg = ctx.createLinearGradient(0, 0, 0, H);
  bg.addColorStop(0, rarity.grad[0]); bg.addColorStop(1, rarity.grad[1]);
  ctx.fillStyle = bg; ctx.fillRect(0, 0, W, H);

  const rgb = hexToRgb(accentColor);
  if (rgb) {
    const rg = ctx.createRadialGradient(W/2, 0, 0, W/2, 0, W * 0.65);
    rg.addColorStop(0, `rgba(${rgb},0.13)`); rg.addColorStop(1, 'rgba(0,0,0,0)');
    ctx.fillStyle = rg; ctx.fillRect(0, 0, W, H);
  }
}
```

- [ ] **Step 5: Verify the front export is byte-identical**

Reload, then run:

```js
await document.fonts.ready;
await renderCard('octocat');
const c = await drawCardToCanvas();
const digest = await crypto.subtle.digest('SHA-256', new TextEncoder().encode(c.toDataURL('image/png')));
const hash = [...new Uint8Array(digest)].map(b => b.toString(16).padStart(2, '0')).join('');
JSON.stringify({ identical: hash === localStorage.getItem('devcard_test_front_hash') })
```

Expected: `{"identical":true}`. If `false`, the extraction changed drawing order or state; fix before continuing. Then run `localStorage.removeItem('devcard_test_front_hash'); 'cleaned'`.

- [ ] **Step 6: Implement `drawCardBackToCanvas` and face-aware download**

Add directly after the closing `}` of `drawCardToCanvas()`:

```js
// Positions come from offsetLeft/offsetTop/offsetWidth/offsetHeight, which are layout values
// that ignore the flip and tilt transforms, so back layout changes carry over automatically.
// Fonts, colors, and letter spacing below must still mirror the .battle-stat* / .card-back* CSS.
function drawCardBackToCanvas() {
  const { rarity, accentColor } = currentCardData;
  const back = document.getElementById('cardBack');
  const scale = 3, W = back.offsetWidth, H = back.offsetHeight;
  const canvas = document.createElement('canvas');
  canvas.width  = W * scale;
  canvas.height = H * scale;
  const ctx = canvas.getContext('2d');
  ctx.scale(scale, scale);

  drawCardBase(ctx, W, H, rarity, accentColor);

  function ls(v) { if ('letterSpacing' in ctx) ctx.letterSpacing = v; }
  const box = el => ({ x: el.offsetLeft, y: el.offsetTop, w: el.offsetWidth, h: el.offsetHeight });
  ctx.textBaseline = 'middle';

  // --- Header ---
  const title = document.getElementById('backTitle');
  const tb = box(title);
  ctx.font = '11px "Bebas Neue", sans-serif'; ls('3.3px');
  ctx.fillStyle = rarity.color;
  ctx.fillText(title.textContent, tb.x, tb.y + tb.h / 2); ls('0px');

  const handle = document.getElementById('backHandle');
  const hb = box(handle);
  ctx.font = '11px "Space Mono", monospace'; ls('1.65px');
  ctx.fillStyle = 'rgba(255,255,255,0.45)';
  ctx.textAlign = 'right';
  ctx.fillText(handle.textContent, hb.x + hb.w, hb.y + hb.h / 2);
  ls('0px'); ctx.textAlign = 'left';

  // --- Stat rows ---
  document.querySelectorAll('#battleStats .battle-stat').forEach(li => {
    const label = li.querySelector('.battle-stat-label');
    const bar   = li.querySelector('.battle-stat-bar');
    const score = li.querySelector('.battle-stat-score');
    const raw   = li.querySelector('.battle-stat-raw');

    const lb = box(label);
    ctx.font = '15px "Bebas Neue", sans-serif'; ls('1.8px');
    ctx.fillStyle = '#fff';
    ctx.fillText(label.textContent, lb.x, lb.y + lb.h / 2); ls('0px');

    const bb = box(bar);
    rrPath(ctx, bb.x, bb.y, bb.w, bb.h, 3);
    ctx.fillStyle = 'rgba(255,255,255,0.08)'; ctx.fill();
    const fillW = bb.w * Number(li.dataset.score) / 100;
    if (fillW > 0) {
      rrPath(ctx, bb.x, bb.y, fillW, bb.h, Math.min(3, fillW / 2));
      ctx.fillStyle = accentColor; ctx.fill();
    }

    const sb = box(score);
    ctx.font = '20px "Bebas Neue", sans-serif';
    ctx.fillStyle = '#fff';
    ctx.textAlign = 'right';
    ctx.fillText(score.textContent, sb.x + sb.w, sb.y + sb.h / 2);
    ctx.textAlign = 'left';

    const rb = box(raw);
    ctx.font = '9px "Space Mono", monospace'; ls('0.9px');
    ctx.fillStyle = 'rgba(255,255,255,0.4)';
    ctx.fillText(raw.textContent, rb.x, rb.y + rb.h / 2); ls('0px');
  });

  // --- Footer ---
  const cardNum = document.getElementById('backCardNumber');
  const cb = box(cardNum);
  ctx.font = '9px "Space Mono", monospace'; ls('0.9px');
  ctx.fillStyle = 'rgba(255,255,255,0.25)';
  ctx.textAlign = 'right';
  ctx.fillText(cardNum.textContent, cb.x + cb.w, cb.y + cb.h / 2);
  ls('0px'); ctx.textAlign = 'left'; ctx.textBaseline = 'alphabetic';

  return canvas;
}
```

In `downloadCard()`, replace:

```js
    const canvas = await drawCardToCanvas();
    const link = document.createElement('a');
    link.download = `devcard-${currentCardData.username}.png`;
```

with:

```js
    const flipped = isCardFlipped();
    const canvas = flipped ? drawCardBackToCanvas() : await drawCardToCanvas();
    const link = document.createElement('a');
    link.download = `devcard-${currentCardData.username}${flipped ? '-back' : ''}.png`;
```

- [ ] **Step 7: Run test to verify it passes**

Reload, rerun the Step 2 snippet. Expected:

```json
{"names":["devcard-octocat.png","devcard-octocat-back.png"],"width":1020,"height":1560,"expectedHeight":1560}
```

(`height` equals `expectedHeight`; both are 1560 when the card is its minimum 520px tall.)

- [ ] **Step 8: Visually compare the back export**

Run:

```js
setCardFlipped(true);
// A canvas element, not <img src="data:...">: the app's CSP img-src blocks data: images.
const preview = drawCardBackToCanvas();
preview.id = 'exportPreview';
preview.style.cssText = 'position:fixed;top:10px;left:10px;width:340px;z-index:9999;outline:1px solid red';
document.body.appendChild(preview);
'preview added'
```

Wait 1 second and take a screenshot. Expected: the red-outlined preview at top left matches the on-screen card back. Same header, six rows in the same vertical positions, bars filled to the same proportions in the accent color, raw values beneath, card number bottom right. Small anti-aliasing differences are fine; misplaced or missing elements are not.

Then run `document.getElementById('exportPreview').remove(); setCardFlipped(false); 'cleaned'` and check `mcp__Claude_Browser__read_console_messages` with `onlyErrors: true` for errors from this task.

- [ ] **Step 9: Update docs**

In `CLAUDE.md`, replace the whole **Canvas export** paragraph (it begins "**Canvas export** (`drawCardToCanvas()`)") with:

```markdown
**Canvas export:** SAVE exports the face showing: `drawCardToCanvas()` for the front (`devcard-{u}.png`) or `drawCardBackToCanvas()` for the back (`devcard-{u}-back.png`), both at 3× and sharing `drawCardBase()` for the clipped background. The front renderer hardcodes a layout mirroring the CSS (340×520 base) and reads values from the front DOM plus `currentCardData`, so any change to the front's markup or styling must be mirrored in it manually. The back renderer positions everything from element `offsetLeft`/`offsetTop`/`offsetWidth`/`offsetHeight` (layout values that ignore the flip and tilt transforms) with `textBaseline = 'middle'`, so back layout changes carry over automatically, but font, color, and letter-spacing changes still need mirroring. Set letter spacing on `ctx` (via the local `ls()` helper) before calling `measureText()` or `wrapLines()`. The avatar is loaded with `crossOrigin='anonymous'` via `loadImgCORS()`.
```

In `README.md`, replace the **Download** bullet with:

```markdown
- **Download** — exports whichever face is showing (front, or the battle stats back) as a 3× resolution PNG rendered via Canvas (pixel-perfect match to the on-page card)
```

- [ ] **Step 10: Commit**

```bash
git add index.html CLAUDE.md README.md
git commit -m "Export the showing card face as PNG

SAVE renders the back via drawCardBackToCanvas (layout read from element
offsets) as devcard-{u}-back.png. Background drawing is shared through
drawCardBase; front export output is byte-identical to before.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```
