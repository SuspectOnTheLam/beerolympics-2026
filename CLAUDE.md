# CLAUDE.md

Guidance for Claude Code (and humans) working in this repository.

## What this project is

**2026 Canada Day Beerolympics** — a live, big-screen **leaderboard** for a backyard
"Beerolympics" tournament (Red Team vs. White Team) held on Canada Day. Throw it up
on a TV/projector; it auto-refreshes and pulls scores live from a Google Sheet.

It is a **static, client-side-only site** hosted on **GitHub Pages**. No build step,
no backend in this repo, no dependencies to install — `index.html` is a single
self-contained file (inline CSS/JS + a base64-embedded pixel font) that fetches
JSON from a Google Apps Script.

### Visual style (preserved across the redesign)

Retro 8-bit arcade look: `Press Start 2P` pixel font (**self-hosted, inlined as
base64** so it renders even with no access to Google Fonts), Canada red/white
palette, a brick-ground strip, and a looping pixel-art canoe (`Canoe.png`) that runs
across the bottom. The layout is **responsive**: it fills the viewport on desktop and
reflows to a single stacked column on phones. There is also a **Full Screen** button
(Fullscreen API) for the big-screen presentation.

## Pages

| File | Purpose |
| --- | --- |
| `index.html` | **Main leaderboard / landing page** (the redesign). Who's-winning banner plus two side-by-side team boards (Red & White), each with its total, a team-MVP strip, and a ranked player list. Plus the full-screen deer.supply ad. |
| `beer-pong-bracket_6.html` | Standalone beer pong tournament bracket (Round 1 → Champion). Older page, untouched by the leaderboard redesign; still reads the same Apps Script API. |
| `Canoe.png` | Pixel-art canoe sprite used in the running animation. |

## Data model (important)

The board is a **read-only view of a Google Sheet** via a Google Apps Script web app
(`API_URL`). **The only data we care about is each player's `name`, `score`, and
which `team` they're on** — read from the API's `redPlayers[]` and `whitePlayers[]`
arrays (each entry `{name, pts}`).

- **Team totals are computed by SUMMING player scores** in `render()`. The sheet's
  pre-computed `winnerName`/`winnerPts`/`loserPts` fields are intentionally **ignored**
  (they were unreliable).
- An optional **bonus CSV** (`BONUS_CSV_URL`, a published-sheet `gviz` CSV) can add
  points to individual players by normalized name. It degrades gracefully — if the
  sheet is unreachable, base scores are shown.
- Scores are parsed leniently (`"18 pts"` → `18`).

If scores need to change, edit the **Google Sheet / Apps Script**, not this repo.

## Leaderboard logic

- **Two boards:** the Red team and White team each have their own board, sorted by
  score descending and ranked independently. Ranks use *competition ranking* (equal
  scores share a rank, e.g. `1, 1, 3`); the top 3 ranks get gold/silver/bronze accents.
- **Who's winning:** a colored banner ("RED TEAM LEADS BY N PTS" / "DEAD HEAT — TEAMS
  TIED AT N") plus a crown 👑, gold glow, and "LEADING" tag on the leading team's board.
- **MVP is per team:** each board's top scorer is that team's MVP — shown in a gold
  `🏆 <TEAM> MVP` strip and badged with `⭐ MVP` + 🏆 in their row.
- **Ties are first-class:** if multiple players tie for a team's top score, that team's
  MVP strip shows an obvious `⚔️ N-WAY TIE` and lists everyone; each tied player gets
  the trophy and an `⭐ MVP · TIE` badge.

## Full-screen ad (deer.supply)

In **full-screen mode only**, a `deer.supply` sponsor ad (fictional GLP-1 weight-loss
brand) overlays the screen **every 5 minutes for 30 seconds** (`AD_EVERY` / `AD_DUR`),
with a live countdown. The scheduler starts on `fullscreenchange` (entering FS) and is
cleared on exit, so the ad never shows in windowed/responsive mode.

## Running locally

No tooling required — plain static files. GitHub Pages serves this repo as a plain
static site (no Jekyll config present), so any static server rooted at the repo is a
faithful local mirror:

```bash
python3 -m http.server 8000
# open http://localhost:8000/   (leaderboard)
```

### Caveats when running locally
- **Live data needs network access to Google** (`script.google.com`). If blocked, the
  status dot turns red ("Failed to fetch") and totals stay `—`; the layout still renders.
  To preview with data, mock the API response (e.g. Playwright `page.route`).
- The fetch **fires immediately on load** (the `30000`/`REFRESH_MS` is only the
  refresh interval), so data appears after one Apps Script round-trip — typically ~1–3s.
- The pixel font is embedded, so it renders offline (no Google Fonts dependency).

## Deployment

- **Host:** GitHub Pages, served from the repository root of `main`.
- **URL:** `https://suspectonthelam.github.io/beerolympics-2026/`
- Pushing to `main` is the whole deploy — the `pages-build-deployment` workflow runs
  automatically; there is no other build or CI step.

## Conventions / gotchas for edits

- `index.html` is **self-contained**: CSS in a `<style>` block, JS in a `<script>`
  block, and the font inlined as a `@font-face` base64 `woff2`. Theme colors are
  `:root` CSS variables (`--red`, `--white`, `--dark`, `--gold`, …).
- Keep the API contract in sync with the Apps Script: the renderer only needs
  `redPlayers[]` / `whitePlayers[]` with `{name, pts}`.
- All user-supplied names are HTML-escaped (`esc()`) before insertion.
