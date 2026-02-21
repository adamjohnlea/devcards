# Dev Cards

Generate collectible trading cards for any GitHub developer. Enter a username, get a card — complete with rarity tier, special ability, language badges, and account vintage. Save it as a high-res PNG or share it on X.

## Features

- **Rarity system** — cards are ranked COMMON, UNCOMMON, RARE, EPIC, or LEGENDARY based on stars and followers
- **Special ability** — generated from the developer's GitHub profile (languages, repos, follower count, hireable status)
- **Account vintage** — shows the year the account was created; accounts 10+ years old earn a VETERAN modifier
- **GitDex** — save cards to a persistent local collection (stored in `localStorage`), click any tile to regenerate
- **Share to X** — opens a pre-filled tweet with the card's rarity and a shareable link
- **Download** — exports a 3× resolution PNG rendered via Canvas (pixel-perfect match to the on-page card)
- **Deep links** — any card URL (`?user=torvalds`) auto-generates the card on load and can be shared directly

## Running

No build step, no dependencies, no package manager. Serve `index.php` with any web server:

```bash
php -S localhost:8000
```

Or drop it into [Laravel Herd](https://herd.laravel.com), Apache, Nginx — anything that serves files over HTTP.

## How It Works

The entire app is a single `index.php` file containing HTML, CSS, and vanilla JavaScript. Despite the `.php` extension it contains no PHP code.

On card generation, two GitHub API requests run in parallel:
- `GET /users/{username}` — profile data
- `GET /users/{username}/repos?per_page=100&sort=updated` — repository data

Stats are computed client-side, a rarity tier is assigned, and the card DOM is updated with a CSS reveal animation.

**Rarity scoring** (`stars + followers × 2`):

| Tier | Score |
|------|-------|
| LEGENDARY | 50,000+ |
| EPIC | 5,000+ |
| RARE | 500+ |
| UNCOMMON | 50+ |
| COMMON | < 50 |

## GitHub API

Uses only public endpoints — no authentication or API key required. Anonymous rate limit is 60 requests/hour per IP. The app surfaces friendly error messages for missing users and API failures.

## Sharing

When a card is generated, the URL updates to `?user=username`. Anyone who opens that link lands directly on that developer's card. The X share button includes the URL so recipients can generate their own card.

## License

GPL-3.0 — see [LICENSE](LICENSE) for details.
