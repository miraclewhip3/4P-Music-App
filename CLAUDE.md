# 4P Music App / Interval Grid

Single-file static HTML app: [interval-grid.html](interval-grid.html). No build step, no dependencies. All HTML/CSS/JS lives in that one file.

## What it is

"Interval Grid" is a time-based musical notation tool for keyboard (RH/LH) voicings — a piano-roll-style grid, not an HTML `<table>`. Notation syntax is deliberately modeled on LilyPond's duration-persistence input style (see the in-app "Notation Key" legend, which is the authoritative, detailed spec — keep it in sync with the parser whenever the grammar changes).

Key departure from LilyPond: **no automatic/relative octave placement.** Octave marks (`'`/`,`) always shift from a lane's fixed default octave, never chained from the previous note. This was an explicit user decision — do not reintroduce auto-octave behavior.

## Running locally

Must be served over `http://`, not opened as `file://` — the page `fetch()`s `styles.json` for the built-in style library, which browsers block for local files. A `FALLBACK_STYLES` object embedded in the JS is used only if that fetch fails.

```bash
python -m http.server 8420
```

`.claude/launch.json` is already configured for this (`static-server`, port 8420) for the Browser pane / `preview_start`.

## Architecture (interval-grid.html)

- **Parser**: `parseHand(str, isRH, err)` — single-pass tokenizer over one hand's full token stream (all bars). Handles bar checks (`|`), `\tuplet 3/2 { a b c }`, `\repeat N { tokens }` (a brace-block consumer, `consumeBlock`), and note/chord/rest tokens. Duration persists across tokens (and across bar lines) until explicitly changed — this is the core LilyPond-derived behavior.
- **Scale steps** (not LilyPond — a user-designed extension): a position may carry `>`/`>>` (up one/two) or `<`/`<<` (down one/two) after its octave marks and before its duration (`1'>>8.`). Steps move along the major scale built on the root (`MAJOR_SCALE`, via `stepScale()`), so with root C, `1>>` = E and `4>` = the C above. Max two per note (error otherwise). Works in chords because `CHORD_RE` is greedy (`<1>>23>8`), and in tuplets. Notes built by `makeNote()` carry `oct` (octave marks) and `steps`; the block stays in its lane's row but is shifted vertically 1/8 cell per step (`stepShift()`: up for `>`, down for `<`; tie bridges follow), and `stepArrows()` draws one full-cell-height arrow per step (↑ for `>`, ↓ for `<`), counter-shifted so the arrows stay exactly on the grid square — that display was an explicit user spec.
- **Units**: `BASE = 32` (32nd-note units per 4/4 bar) is the resolution bar-check math is done in. Each bar's total must sum to exactly 32 or a warning prints.
- **Rendering**: absolutely-positioned "lane-row" strips built by `buildTimeline()` / `buildHandRows()` — one continuous horizontal timeline across all displayed bars (1–4, side by side), with thin `.bar-divider` lines at bar boundaries. No per-bar headings.
- **Grid cells are ALWAYS square. ALWAYS.** One column (the finest gridline spacing) is exactly as wide as a lane row is tall (`CELL = 40`, matching `.lane-row` height in CSS), no matter whether columns represent quarter, 8th, or 16th notes. Never stretch the grid to fill the panel or shrink it to fit — a wide grid scrolls horizontally instead. (Both were tried and rejected: fit-to-width stretched 8th cells into wide rectangles; the old fixed 7px/unit squished them into narrow ones.)
- **Column size**: `colUnits` (8=quarter, 4=8th, 2=16th) is chosen per render by `columnUnits(rh, lh)` — the coarsest of those that every event's `startUnits` and `durUnits` in both hands is a multiple of, so no block starts or ends mid-column. Only quarters → quarter columns; any 8th or dotted quarter → 8ths; any 16th, dotted 8th, tuplet, or 32nd → 16ths (finest shown). `PX` (px per unit) = `CELL / colUnits`. Gridlines are set inline by `gridlines()` (beat lines always; 8th/16th lines only down to `colUnits`) so they track `PX`. Because the scale is only known after both hands are parsed, `\tuplet` events store fractional `startUnits`/`durUnits` (resolved to pixels at render time) rather than precomputed pixel offsets.
- **Styles/presets**: `STYLES` = merge of `BUILTIN_STYLES` (fetched from `styles.json`, falls back to embedded `FALLBACK_STYLES`) and `customStyles` (persisted to `localStorage` key `intervalGridStyles`). Each style: `{tempo, key, chordProgression, tags, swing, rh, lh, variationOf}` — `bars` is not stored, it's derived from the notation at render time. `tags` is an optional array of strings; the Filter by Tag dropdown (`filteredStyleNames()`) narrows the Style Name list to styles carrying the selected tag. `variationOf` links a style to a non-variation parent — variations can't nest, and `rebuildVariationOptions()` / `renderVariationsFor()` keep the parent dropdown and child chip list in sync.

## styles.json

Version-controlled built-in style library, fetched at runtime — this is the file to hand-edit when adding/fixing styles (not the embedded `FALLBACK_STYLES` copy in the HTML, which is just an offline fallback and should stay minimal). Several entries (Dance 1/2, Basic 1-4) were transcribed by best-effort reading of OneMotion Chord Player screenshots and are flagged as needing verification against the source tool — check before trusting them as ground truth.

When editing `rh`/`lh` strings by hand: duration persists after rests too (`r8.` sets the running duration to dotted-8th for whatever comes next, including a bare position token) — re-specify the duration explicitly if you don't want it inherited. Verify bar totals sum to exactly 32 units before committing (a quick Node script that reuses the parser's unit logic is faster than eyeballing it).

## Workflow notes

- **At the start of every session, before anything else, do both of these on the first turn without being asked:**
  1. **Sync git**: `git fetch`, then fast-forward `master` to `origin/master` (`git pull --ff-only`). If there are uncommitted local changes, stash them first and `git stash pop` after; if the pop conflicts or the branch has diverged, stop and tell the user rather than resolving it silently. Report what came in.
  2. **Start the local server**: `preview_start` with `static-server` (reuses it if already up), then navigate the tab to `http://localhost:8420/interval-grid.html`.
- `git` is not on PATH on this machine — prepend GitHub Desktop's bundled git (`C:\Users\Patrick\AppData\Local\GitHubDesktop\app-*\resources\app\git\cmd`, newest `app-*` version) and `C:\Program Files\GitHub CLI` to `$env:Path` in each PowerShell call.

- **Always commit and push after each completed change in this repo** — do this by default, without asking first.
- `gh` CLI is authenticated as `miraclewhip3`; remote is `https://github.com/miraclewhip3/4P-Music-App` (private). If `git push` fails with "Repository not found" despite `gh auth status` looking fine, run `gh auth setup-git` (stale git credential helper, not a gh auth problem).
- For parser/grammar changes, verify with a standalone Node test (extract the parsing logic, no DOM) before wiring into the browser — this has caught real bugs (duration-persistence math) that were easy to miss by inspection.
