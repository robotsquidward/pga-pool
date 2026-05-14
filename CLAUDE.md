# pga-pool

Static single-page scoreboard for a friends' golf-major pool. One file: `index.html` (HTML + CSS + JS inline). Deployed via GitHub Pages. No build step, no dependencies.

Live scores come from the public ESPN endpoint:
`https://site.api.espn.com/apis/site/v2/sports/golf/leaderboard?league=pga`

## Recurring task: update for the next major

Each major, the owner drops a fresh `PGA Pools.xlsx` in `~/Downloads/` and asks for the same site re-skinned for the new tournament. The workbook has one tab per tournament named `<year> <tournament>` (e.g. `2026 PGA`, `2026 Masters`, `2025 The Open`, `2025 USA`).

### Sheet schema (all tournament tabs)
Row 1 is a header. Rows 2+ are participants.

| A    | B–M (alternating)                          | N           |
|------|--------------------------------------------|-------------|
| Name | Golfer 1, Score, Golfer 2, Score, … (×6)   | Total Score |

Only columns A, B, D, F, H, J, L matter for the site — the Score/Total columns are the host's post-tournament tally, not used by the live scoreboard. The site recomputes points live from ESPN.

### Reading the xlsx without openpyxl
`pip install` is blocked by PEP 668 on this machine. Unzip the xlsx and parse the XML directly:

```bash
unzip -o "/Users/ajkueterman/Downloads/PGA Pools.xlsx" -d /tmp/pga_xlsx
# Sheet name → file: grep workbook.xml for the sheet, then map r:id via
# xl/_rels/workbook.xml.rels to xl/worksheets/sheetN.xml.
# String cells (t="s") index into xl/sharedStrings.xml.
```

A small Python ET script that walks `<row>`/`<c>` and resolves shared strings + inline strings is the fastest path. See the prior conversation history if you need a template — but writing it fresh is ~20 lines.

### What to change in `index.html`
1. `<title>` and `<h1>` — tournament name.
2. `.header-sub` — course + dates.
3. `:root` color variables + accent classes — re-skin to match the major. Past conventions:
   - **Masters**: greens + gold (`--green-dark`, `--gold`, cream bg).
   - **PGA Championship**: navy/blue/white/silver.
   - **U.S. Open**: red/white/navy (USGA palette).
   - **The Open**: claret + tan/cream, or tartan-inspired.
4. `const DRAFT = [...]` — replace with the new draft. Preserve sheet row order; the UI uses it as the stable tiebreaker.
5. Leave scoring, ESPN fetch, normalize(), and render logic alone unless asked.

### Name normalization gotchas
`normalize()` strips diacritics (NFD + combining-marks range) and has explicit ASCII fallbacks for letters that don't decompose (`ø`, `ð`, `þ`, `ß`). When copying names from the sheet into `DRAFT`:
- Either keep the diacritics (e.g. `Højgaard`) — normalize() handles them.
- Or pre-strip to ASCII (`Hojgaard`) — also fine.
- Watch for the sheet's typos (e.g. `Joe HIghsmith` in the 2026 PGA sheet) — write the correct casing in `DRAFT` since ESPN's feed won't match the typo.
- `(a)` amateur tags from ESPN are stripped automatically.

### Scoring rule
`points = max(0, 10 − scoreToPar)`, zero for CUT or WD. Defined in `poolPoints()`. Don't change unless the owner asks — the rule is league-stable across majors.
