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
across a dedicated bottom "scene" band (`--lane`, reserved as the app's bottom padding)
so the boat is drawn in front of — and never clipped by — the team boards. The layout is **responsive**: it fills the viewport on desktop and
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

- **Team totals come DIRECTLY from the Google Sheet** — the `winnerPts`/`loserPts`
  fields, mapped to red/white by which of `winnerName`/`loserName` contains
  "red"/"white" (see `sheetTeamTotals()`). This is deliberate: **a team can earn
  points without crediting any individual player**, so the sheet's team number can be
  *greater* than the sum of player scores, and we respect the sheet as authoritative.
  Summing player scores is only a **fallback** when the sheet omits a team number.
  (Player rows and the per-team MVP still use individual player scores.)
- An optional **bonus CSV** (`BONUS_CSV_URL`, a published-sheet `gviz` CSV) can add
  points to individual players by normalized name. It degrades gracefully — if the
  sheet is unreachable, base scores are shown.
- Scores are parsed leniently (`"18 pts"` → `18`).

If scores need to change, edit the **Google Sheet / Apps Script**, not this repo.

## Leaderboard logic

- **Two boards:** the Red team and White team each have their own board, sorted by
  score descending and ranked independently. Ranks use **dense ranking** (equal scores
  share a rank, distinct scores step by 1 with no gaps, e.g. `1, 1, 2`) — chosen over
  competition ranking so the numbers never appear to "skip". The same dense logic is
  used in `syncRows` and `snapshot` so they stay consistent. Top 3 ranks get
  gold/silver/bronze accents.
- **Who's winning:** a colored banner ("RED TEAM LEADS BY N PTS" / "DEAD HEAT — TEAMS
  TIED AT N") plus a crown 👑, gold glow, and "LEADING" tag on the leading team's board.
- **Dynamic page theme:** the whole page re-themes to follow the leader — `theme-red`
  (deep-red page) when Red leads, `theme-white` (cream page) when White leads, and
  `theme-tie` (yellow page) on a dead heat. Theme-able colors (page/scene background,
  frame, title, subtitle, status) are CSS variables overridden per `body.theme-*` class,
  chosen for contrast against each background; team boards keep their own colors.
- **MVP is per team:** each board's top scorer is that team's MVP — shown in a gold
  `🏆 <TEAM> MVP` strip and badged with `⭐ MVP` + 🏆 in their row.
- **Ties are first-class:** if multiple players tie for a team's top score, that team's
  MVP strip shows an obvious `⚔️ N-WAY TIE` and lists everyone; each tied player gets
  the trophy and an `⭐ MVP · TIE` badge.

## Animations

The board is built to animate between refreshes, so rows must keep a stable identity
across renders. `syncRows` therefore does a **keyed reconcile** (one persistent DOM row
per player name, kept in `container._rows`) instead of rebuilding `innerHTML`, and uses
the **FLIP** technique (measure First positions → reorder DOM → measure Last → play an
inverse `el.animate` transform) so rows slide to their new rank when the order changes.
On top of that: team totals and player scores **count up**, a row **flashes** green/red
when a score changes (with a score "bump"), a floating **▲/▼ rank-delta** badge appears
when a player moves, the MVP strip **pops** when the MVP changes, the lead banner pops
when the lead changes hands, and the theme cross-fades. Idle flourishes: bobbing crown,
pulsing trophies + LEADING tag + winning-board ring, and a gently bobbing canoe. All
motion is gated behind `prefers-reduced-motion`.

## deer.supply ad — 8-bit video segment

A `deer.supply` sponsor ad (a **fictional** GLP-1/triple-agonist "RETA-DEER™" weight-loss
brand, shown for entertainment only — see disclaimers in the ad copy) overlays the screen
**every 5 minutes for 30 seconds** (`AD_EVERY` / `AD_DUR`). It plays on **desktop
regardless of fullscreen** (`syncAds()` runs the scheduler at desktop width; skipped on
phone-width layouts), not only in TV mode.

It's a **4-scene animated "video"** driven by a `requestAnimationFrame` timeline:
intro (pixel-art 8-bit deer + logo) → hero transformation (an 8-bit person slims from a
high BMI to healthy as a `BMI`/`MONTH 0→18`/progress bar advance) → crowd results
(`24% BODY WEIGHT GONE*`, a dramatization of published retatrutide trial figures) → CTA.
Sprites are real pixel art (deer rendered from a char matrix into a CSS grid; people are
blocky CSS figures whose belly width animates via an `@property --belly` length). A Web
Audio **chiptune** plays during the ad (unlocked on the first user gesture). Skippable by
tap/space; auto-closes at 30s; CRT scanline + vignette overlay.

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
