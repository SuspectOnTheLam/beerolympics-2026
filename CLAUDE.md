# CLAUDE.md

Guidance for Claude Code (and humans) working in this repository.

## What this project is

**2026 Canada Day Beerolympics** — a set of live, big-screen scoreboard pages for a
backyard "Beerolympics" tournament (Red Team vs. White Team) held on Canada Day.
The pages are designed to be thrown up on a TV/projector and left running; they
auto-refresh and pull their data live from a Google Sheet.

It is a **static, client-side-only site** hosted on **GitHub Pages**. There is no
build step, no backend in this repo, and no dependencies to install — just HTML +
inline CSS/JS that fetch JSON/CSV from Google.

### Visual style

Retro 8-bit arcade look: `Press Start 2P` pixel font (loaded from Google Fonts),
Canada red/white palette, a brick-ground strip, a maple leaf, and a looping
pixel-art canoe (`Canoe.png`) that runs across the bottom. Every page is authored
on a fixed **1920×1080** "stage" that is scaled to fit the viewport via JavaScript
(`scaleStage()`), so it always fills a TV regardless of resolution.

## Pages

| File | Purpose |
| --- | --- |
| `index.html` | **Main scoreboard / landing page.** Shows the current winning team, each team's MVP, and full per-player point lists for Red and White. Background flips red when Red is winning. |
| `beer-pong-bracket_6.html` | **Beer pong tournament bracket.** Single-elimination bracket (Round 1 → Quarterfinals → Semifinals → Champion) with SVG connector lines drawn between rounds. |
| `Canoe.png` | Pixel-art canoe sprite used in the running animation on both pages. |
| `README.md` | Minimal repo readme. |

## Data flow (important)

Both pages are **read-only views of a Google Sheet**. They do not store state.

- **Live scores + bracket** come from a Google Apps Script Web App:
  `API_URL = https://script.google.com/macros/s/AKfycbz…/exec`
  - `index.html` expects JSON: `winnerName`, `winnerPts`, `loserName`, `loserPts`,
    `redMvp{name,pts}`, `whiteMvp{name,pts}`, `redPlayers[]`, `whitePlayers[]`.
  - `beer-pong-bracket_6.html` expects JSON with a `bracket` object:
    `r1[]` (8 team slots), `r2[]`, `r3[]`, `r4[]` (champion).
- **Bonus points overlay** (`index.html` only) come from a published Google Sheet
  CSV (`BONUS_CSV_URL`, the `gviz/tq?...out:csv` endpoint). Bonuses are matched to
  players by normalized name and added on top of the main scores. If the bonus
  sheet is unreachable, the page degrades gracefully and shows main scores only.
- Both pages poll every **30s** (`REFRESH_MS = 30000`) and show a live/fetching/error
  status dot.

If scores need to change, edit the **Google Sheet / Apps Script**, not this repo.
The only reason to redeploy here is to change layout, styling, or the data contract.

## Running locally

No tooling required — it's plain static files. GitHub Pages serves this repo as a
plain static site (no Jekyll config is present), so any static file server rooted
at the repo gives a faithful local mirror. Serve from the repo root so `Canoe.png`
resolves:

```bash
# Python (matches GitHub Pages static serving; index.html is the default doc)
python3 -m http.server 8000
# then open http://localhost:8000/                       (scoreboard)
#           http://localhost:8000/beer-pong-bracket_6.html (bracket)
```

### Caveats when running locally
- **Live data needs network access to Google.** `script.google.com` and
  `docs.google.com` must be reachable, or the status dot turns red and pills show
  `—`. The layout still renders.
- **The pixel font** loads from `fonts.googleapis.com`; if blocked, the page falls
  back to a generic monospace and looks slightly off but works.
- Open the page **full-screen** to see the intended 1920×1080 scaled layout.

## Deployment

- **Host:** GitHub Pages, served from the repository root of the default branch.
- **Staging/production URL:** `https://suspectonthelam.github.io/beerolympics-2026/`
- Deploys happen automatically on push to the Pages-configured branch — there is no
  build or CI step. Changing a file and pushing is the whole deploy.

## Conventions / gotchas for edits

- Each page is **self-contained**: CSS in a `<style>` block and JS in a `<script>`
  block in the same file. There is no shared stylesheet or JS module — if you change
  shared visuals (colors, font, canoe, brick ground), update **both** HTML files.
- Theme colors live in `:root` CSS variables (`--red`, `--white`, `--dark`, etc.).
- The `.stage` is fixed at 1920×1080; size things in px against that canvas, not the
  viewport. `scaleStage()` handles fitting it to the screen.
- Keep the data contract in sync with the Apps Script. The expected JSON shape is
  documented in a comment block in `index.html` (`render(d)`).
- Team color is inferred from team labels/names (e.g. team number ≥ 5 ⇒ Red in the
  bracket; `winnerName` containing "red" flips the scoreboard background).
