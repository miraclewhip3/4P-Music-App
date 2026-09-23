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
- **Units**: `BASE = 32` (32nd-note units per 4/4 bar) is the resolution bar-check math is done in. Each bar's total must sum to exactly 32 or a warning prints.
- **Rendering**: absolutely-positioned "lane-row" strips built by `buildTimeline()` / `buildHandRows()` — one continuous horizontal timeline across all displayed bars (1–4, side by side), with thin `.bar-divider` lines at bar boundaries. No per-bar headings.
- **Pixel scale**: mutable `PX` (px per unit) is recomputed every render (and on window resize) to stretch the timeline to fill the grid panel: `PX = max(minPX, floor(availableWidth / (bars*BASE)))`. `minPX` is `PX_BASE = 7`, doubled when `finestUnits(rh, lh)` finds anything finer than an 8th (< 4 units) in either hand (`fine`); below that the panel scrolls horizontally instead of squishing notes. Gridlines are set inline by `gridlines()` (beat + 8th lines; 16th lines only when `fine`) so they track `PX`. Because the scale is only known after both hands are parsed, `\tuplet` events store fractional `startUnits`/`durUnits` (resolved to pixels at render time) rather than precomputed pixel offsets.
- **Styles/presets**: `STYLES` = merge of `BUILTIN_STYLES` (fetched from `styles.json`, falls back to embedded `FALLBACK_STYLES`) and `customStyles` (persisted to `localStorage` key `intervalGridStyles`). Each style: `{tempo, key, chordProgression, swing, bars, rh, lh, variationOf}`. `variationOf` links a style to a non-variation parent — variations can't nest, and `rebuildVariationOptions()` / `renderVariationsFor()` keep the parent dropdown and child chip list in sync.

## styles.json

Version-controlled built-in style library, fetched at runtime — this is the file to hand-edit when adding/fixing styles (not the embedded `FALLBACK_STYLES` copy in the HTML, which is just an offline fallback and should stay minimal). Several entries (Dance 1/2, Basic 1-4) were transcribed by best-effort reading of OneMotion Chord Player screenshots and are flagged as needing verification against the source tool — check before trusting them as ground truth.

When editing `rh`/`lh` strings by hand: duration persists after rests too (`r8.` sets the running duration to dotted-8th for whatever comes next, including a bare position token) — re-specify the duration explicitly if you don't want it inherited. Verify bar totals sum to exactly 32 units before committing (a quick Node script that reuses the parser's unit logic is faster than eyeballing it).

## Workflow notes

- **Always commit and push after each completed change in this repo** — do this by default, without asking first.
- `gh` CLI is authenticated as `miraclewhip3`; remote is `https://github.com/miraclewhip3/4P-Music-App` (private). If `git push` fails with "Repository not found" despite `gh auth status` looking fine, run `gh auth setup-git` (stale git credential helper, not a gh auth problem).
- For parser/grammar changes, verify with a standalone Node test (extract the parsing logic, no DOM) before wiring into the browser — this has caught real bugs (duration-persistence math) that were easy to miss by inspection.
