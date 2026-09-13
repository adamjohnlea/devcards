# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the Application

Zero-dependency, single-file app: everything (HTML, CSS, vanilla JS) lives in `index.html`. No build step, package manager, linter, or test suite. Serve it over HTTP with anything:

```bash
php -S localhost:8000  # or Laravel Herd, Apache, Nginx
```

Load `?user=<username>` to jump straight to a card (the init block at the bottom of the script auto-renders it).

## Architecture

**Card generation flow** (`renderCard(username)`, shared by the input, quick-gen buttons, GitDex tiles, and deep links):
1. Username validated against `VALID_USERNAME` regex (also used to sanitize GitDex entries and the `?user=` param)
2. `loadCardData()` returns a fresh `localStorage` cache entry if one exists; otherwise `fetchUserData()` fetches `GET /users/{u}` plus repo page 1 in parallel, then pages 2..ceil(`public_repos`/100) in parallel, then keeps fetching while a page comes back full (guards against a stale `public_repos`). Repos use `sort=full_name` because `sort=updated` can shift repos across page boundaries mid-fetch. `githubHeaders()` adds a Bearer token if one is saved. Any failed request throws via `apiError()` (404 → "User not found", 403/429 → rate-limit message with reset time and a token hint); never substitute partial repo data, since it would be cached
3. `computeStats(repos)` → total stars + languages sorted by frequency
4. `calcRarity(stars, followers)` → tier label, color, gradient. Score is `stars + followers*2`: LEGENDARY > 50,000 | EPIC > 5,000 | RARE > 500 | UNCOMMON > 50 | COMMON ≤ 50 (strict `>`, so a boundary score drops to the lower tier)
5. `buildAbility(user, langs)` → special ability text
6. Card DOM populated, `currentCardData` set, URL updated via `history.replaceState`, VETERAN modifier shown if account ≥ 10 years old

**Canvas export** (`drawCardToCanvas()`): a hand-written Canvas renderer (not html2canvas) that redraws the card at 3× (340×520 base). It reads most values back **from the rendered card DOM** plus `currentCardData`, so any change to the card's markup or styling must be mirrored manually in this function to keep the PNG matching. Set letter spacing on `ctx` (via the local `ls()` helper) before calling `wrapLines()` so `measureText()` wraps correctly. The avatar is loaded with `crossOrigin='anonymous'` via `loadImgCORS()`.

**Persistence (`localStorage`):**
- `gitdex`: array of saved cards `{username, name, avatarUrl, rarityColor, rarityLabel, ...}`. `getDex()` strictly validates every entry (username regex, avatar must be on `avatars.githubusercontent.com`, hex color, known rarity label) and drops invalid ones; keep that validation in sync when adding fields. `renderGitDex()` handles tile rendering plus mouse and touch drag-to-reorder.
- `gh_token`: optional GitHub PAT from the settings panel (gear icon), raising the rate limit from 60 to 5,000 req/hr.

**Styling:** theming via CSS custom properties (`--bg`, `--surface`, `--border`, `--accent`, `--text`, `--muted`). Card effects use 3D perspective transforms (mouse parallax set in `renderCard`), clip-path, and CSS animations (`cardReveal`, `foilSweep`, `spin`). The `.card-foil` overlay is a pointer-tracked glare: the same mousemove handler that tilts the card sets `--foil-x`, `--foil-y`, and `--foil-shift` on `.card`, and `foilSweep` plays one shine pass on reveal (disabled under `prefers-reduced-motion`). The foil is screen-only and intentionally not drawn by the PNG export. Fonts: Bebas Neue (headings), Space Mono (mono), Rajdhani (UI), loaded from Google Fonts. Card accent color comes from the top language via `getLangColor()`/`LANG_COLORS` (add new languages there); a language missing from the map gets `#888`, and the rarity color is used only when the user has no languages at all.

## Content Security Policy

A strict CSP `<meta>` tag sits in `<head>`. Any new external resource must be allowlisted there or it will be blocked silently. Currently allowed: scripts from `googletagmanager.com` (Google Analytics); styles from `fonts.googleapis.com`; fonts from `fonts.gstatic.com`; images from `*.githubusercontent.com`; fetches to `api.github.com` and the Google Analytics endpoints. `frame-src` and `object-src` are `'none'`.
