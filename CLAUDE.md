# 4P Music App / Interval Grid

Single-file static HTML app: [interval-grid.html](interval-grid.html). No build step, no dependencies. All HTML/CSS/JS lives in that one file.

## What it is

"Interval Grid" is a time-based musical notation tool for keyboard (RH/LH) voicings — a piano-roll-style grid, not an HTML `<table>`. Notation syntax is deliberately modeled on LilyPond's duration-persistence input style (see the in-app "Notation Key" legend, which is the authoritative, detailed spec — keep it in sync with the parser whenever the grammar changes).

**The most important part of a style is its RH and LH pattern.** Every other attribute (tempo, key, feel, chord progression) is a default that Automatic mode can override. E.g. Basic 2 is written in C Major at 100 BPM, but in Automatic mode with a higher Key level it may be played in D Major over a generated progression. Positions are therefore chord tones (root/3rd/5th/7th), never fixed pitches — keep patterns key- and chord-independent.

Key departure from LilyPond: **no automatic/relative octave placement.** Octave marks (`'`/`,`) always shift from a lane's fixed default octave, never chained from the previous note. This was an explicit user decision — do not reintroduce auto-octave behavior.

## Running locally

Must be served over `http://`, not opened as `file://` — the page `fetch()`s `styles.json` (with `cache:'no-cache'`, so edits show on reload) for the built-in style library, which browsers block for local files. A `FALLBACK_STYLES` object embedded in the JS is used only if that fetch fails. The page must keep its `<meta charset="utf-8">`: the server sends no charset, and without it the browser decodes the script's `–`, `·`, `°` as Windows-1252.

```bash
python -m http.server 8420
```

`.claude/launch.json` is already configured for this (`static-server`, port 8420) for the Browser pane / `preview_start`.

## Architecture (interval-grid.html)

- **Parser**: `parseHand(str, isRH, err)` — single-pass tokenizer over one hand's full token stream (all bars). Handles bar checks (`|`), `\tuplet 3/2 { a b c }`, `\repeat N { tokens }` (a brace-block consumer, `consumeBlock`), and note/chord/rest tokens. Duration persists across tokens (and across bar lines) until explicitly changed — this is the core LilyPond-derived behavior.
- **Scale steps** (not LilyPond — a user-designed extension): a position may carry `>`/`>>` (up one/two) or `<`/`<<` (down one/two) after its octave marks and before its duration (`1'>>8.`). Steps move along the chord's scale (`chordScale()` — see Music theory below), so on C in C Major `1>>` = E and `4>` = the C above. Max two per note (error otherwise). Works in chords because `CHORD_RE` is greedy (`<1>>23>8`), and in tuplets. Notes built by `makeNote()` carry `tone` (0=root, 1=3rd, 2=5th, 3=7th — from the lane), `oct` (octave marks) and `steps`, but no pitch: pitch is resolved at render time by `realizeNote()`; the block stays in its lane's row but is shifted vertically 1/8 cell per step (`stepShift()`: up for `>`, down for `<`; tie bridges follow), and `stepArrows()` draws one full-cell-height arrow per step (↑ for `>`, ↓ for `<`), counter-shifted so the arrows stay exactly on the grid square — that display was an explicit user spec.
- **Units**: `BASE = 32` (32nd-note units per 4/4 bar) is the resolution bar-check math is done in. Each bar's total must sum to exactly 32 or a warning prints.
- **Rendering**: absolutely-positioned "lane-row" strips built by `buildTimeline()` / `buildHandRows()` — one continuous horizontal timeline across all displayed bars (1–4, side by side), with thin `.bar-divider` lines at bar boundaries. No per-bar headings.
- **Grid cells are ALWAYS square. ALWAYS.** One column (the finest gridline spacing) is exactly as wide as a lane row is tall (`CELL = 40`, matching `.lane-row` height in CSS), no matter whether columns represent quarter, 8th, or 16th notes. Never stretch the grid to fill the panel or shrink it to fit — a wide grid scrolls horizontally instead. (Both were tried and rejected: fit-to-width stretched 8th cells into wide rectangles; the old fixed 7px/unit squished them into narrow ones.)
- **Column size**: `colUnits` (8=quarter, 4=8th, 2=16th) is chosen per render by `columnUnits(rh, lh)` — the coarsest of those that every event's `startUnits` and `durUnits` in both hands is a multiple of, so no block starts or ends mid-column. Only quarters → quarter columns; any 8th or dotted quarter → 8ths; any 16th, dotted 8th, tuplet, or 32nd → 16ths (finest shown). `PX` (px per unit) = `CELL / colUnits`. Gridlines are set inline by `gridlines()` (beat lines always; 8th/16th lines only down to `colUnits`) so they track `PX`. Because the scale is only known after both hands are parsed, `\tuplet` events store fractional `startUnits`/`durUnits` (resolved to pixels at render time) rather than precomputed pixel offsets.
- **Styles/presets**: `STYLES` = merge of `BUILTIN_STYLES` (fetched from `styles.json`, falls back to embedded `FALLBACK_STYLES`) and `customStyles` (persisted to `localStorage` key `intervalGridStyles`). Each style: `{tempo, key, chordProgression, tags, difficulty, swing, rh, lh, variationOf}` — `difficulty` (1–10, default 1) is the pattern difficulty Automatic mode filters on; `bars` is not stored, it's derived from the notation at render time. `tags` is an optional array of strings; the Filter by Tag dropdown (`filteredStyleNames()`) narrows the Style Name list to styles carrying the selected tag. `variationOf` links a style to a non-variation parent — variations can't nest, and `rebuildVariationOptions()` / `renderVariationsFor()` keep the parent dropdown and child chip list in sync.

## Music theory (pure functions, testable with the parser)

- **Spelling** is by letter + accidental (`spell()`, `LETTERS`, `NAT_PC`), never by nearest natural, so keys spell correctly (F# Major has E#°, Gb Major has Cb).
- **Keys**: `KEY_LEVELS` (the difficulty table below); `parseKey('Bb Major')` → `{letter, pc, minor}`. Minor keys use minor-key harmony with harmonic-minor V and vii° (`DIATONIC.minor`).
- **Chords** are `{letter, pc, quality}` (+ optional `suffix` for an unrecognised quality like `add9`, kept verbatim). `QUALITIES` lists each quality's intervals (sus chords put the 3rd lane on the 2nd/4th). `diatonicChord(key, degree, 'triad'|'seventh'|'dom7')` builds chords from scale degrees; `romanFor(key, chord)` names them (I, vi, V7, viiø7, II7…).
- **Numeral conversion**: `parseProgression()` / `formatProgression()` convert the Chord Progression text; `transposeProgression(text, fromKey, toKey)` runs on every manual key change. A chord that is diatonic in the old key keeps its numeral and takes the new key's quality (I–vi–IV–V in C Major → i–VI–iv–V in A Minor); anything else is transposed literally.
- **Pitch realization**: `realizeNote(note, key, chord)` → `{semi, name}`. `chordScale()` is the key's scale starting on the chord root with the chord's own tones substituted in (E major in A minor walks G#; D7 in C walks F#). The lane tone picks a chord tone; for a triad, the 7th lane is the key's 7th above the root. Manual mode realizes every bar against the key's tonic chord (so C Major output is the same as before this existed); Automatic mode uses each bar's chord.

## Modes

- **Manual**: the original workflow — pick/edit/save styles.
- **Automatic**: the refresh button (`generate()`) picks a style, a key and a 4-chord progression (`generateProgression()`), stored in `gen`. `render()` cycles the style's bars across the 4 chords (`cycleBars()`: grid bar b plays style bar b mod styleBars), shows chord labels above the bars, and realizes notes per bar. Settings are disabled except Tempo and Feel; those edits are sticky overrides (`autoOverride`) across refreshes until "reset" or a mode switch. The Editor is collapsed and read-only. Switching back to Manual reloads the style as written. Mode and slider values persist in `localStorage` key `intervalGridMode`.
- **Level rules (user decisions)**: Key = exactly that level. Chords = cumulative up to the level, always including ≥1 chord from that level, starting on the tonic, no back-to-back repeat. Patterns = styles with `difficulty` ≤ level, preferring exactly that level; styles with parse errors or incomplete bars (`styleIsPlayable()`) are never picked.
- **Key difficulty**: 1 C · 2 F, G · 3 Bb, D · 4 Eb, A · 5 E, Ab · 6 Db, B · 7 Gb, F# · 8 Dm, Am, Em · 9 Gm, Cm, Fm · 10 Bm, F#m, C#m, Bbm, Ebm, G#m.
- **Chord difficulty** (`CHORD_LEVELS`): 1 I, IV, V · 2 ii, vi, V7 · 3 iii, vii° · 4 diatonic 7ths (Imaj7, ii7, iii7, IVmaj7, V7, vi7, viiø7) + a dominant 7th on any degree (I7, II7, III7…; in minor, rooted on the natural-minor degree). The slider is capped at 4 until levels 5–10 are defined.

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
- For parser/grammar/theory changes, verify with a standalone test (extract the parsing logic, no DOM) before wiring into the browser — this has caught real bugs (duration-persistence math) that were easy to miss by inspection. Node isn't installed on this machine, so run it in the Browser pane: `fetch('interval-grid.html')`, slice from `var BASE = 32` to `var swing` (constants) plus from `function unitsFor` to `// One continuous timeline` (parser + theory), and eval with `new Function(...)`.
