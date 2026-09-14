# AlignLab — Sequence Alignment Lab

A single-file, framework-free web app that visualizes and computes DNA/protein sequence alignment using classic bioinformatics algorithms — Needleman-Wunsch (global) and Smith-Waterman (local) — with live scoring, an interactive DNA helix visualization, algorithm benchmarks, and a printable report.

No build step, no dependencies, no server. Open `AlignLab.html` in Chrome (or any modern browser) and it runs.

---

## 1. Introduction

AlignLab lets you take two biological sequences (or any short strings of letters), pick an alignment algorithm and a scoring scheme, and instantly see how they align — the aligned strings, the score, the number of matches, and where the algorithm made trade-offs between matching characters and inserting gaps.

It's built as a small "console"-style dashboard: a sidebar for navigation and saved sequence pairs, a hero panel with a DNA helix visualization and the live alignment result, a scoring-presets panel, a benchmarks panel comparing five alignment strategies, and a reports page that generates a printable summary of the current analysis.

The entire thing — UI, state management, alignment algorithms, and rendering — lives in one `.html` file with embedded CSS and JavaScript. There's no React, no npm install, no bundler. It's meant to double as both a working tool and a compact demonstration of core CS/algorithms concepts (dynamic programming, string alignment, complexity analysis) wrapped in a polished UI.

---

## 2. What it does (feature list)

- **Global alignment (Needleman-Wunsch)** — aligns two full sequences end-to-end, penalizing gaps anywhere.
- **Local alignment (Smith-Waterman)** — finds the best-scoring matching subsequence, ignoring unrelated flanking regions.
- **Configurable scoring** — four presets (Standard, Strict, Lenient gaps, Affine-style) controlling match reward, mismatch penalty, and gap penalty, or you can think of them as different biological assumptions (e.g. "gaps are cheap" vs "mismatches are costly").
- **Editable sequences** — swap in your own sequences (A–Z letters only, auto-uppercased and capped at 20 characters for the visualization) and see the alignment recompute live.
- **Saved sequence pairs** — four example pairs (including a noisy-motif case and an insulin-like fragment) selectable from the sidebar or a grid of cards.
- **DNA double-helix visualization** — an SVG helix generated from the actual first sequence's bases, colored per base (A/T/G/C/U), not a static decoration.
- **Complexity analytics page** — a chart showing DP-matrix cell growth vs. sequence length (quadratic), with theoretical and empirically "measured" complexity stats.
- **Benchmarks panel** — compares five algorithms/strategies (NW, SW, Affine-gap Gotoh, BLOSUM62-scored, and a star-alignment MSA consensus) side by side, all visually highlighted with their own accent color, with a marker on whichever one matches your currently selected mode.
- **Full report generation** — a Reports page that, only on demand, builds a complete text report (sequences, algorithm, scoring, result, complexity, benchmark table) and opens the browser's native print dialog so it can be printed or saved as a PDF.
- **Dark / light mode** — a single toggle switches the entire UI's color scheme instantly via CSS variables.
- **Copy to clipboard** — one click copies the current aligned sequences.

---

## 3. Tech stack

| Layer | Choice | Why |
|---|---|---|
| Markup/Styling | Plain HTML + CSS (CSS custom properties for theming) | No compile step; CSS variables make theme-switching trivial |
| Logic/State | Vanilla JavaScript (ES6+) | Removes the React/bundler dependency so the file is portable and opens directly in a browser |
| Rendering | Manual `innerHTML` templating + one delegated event listener | Small enough app that a virtual DOM is unnecessary overhead |
| Fonts | Google Fonts (Manrope, JetBrains Mono) via `<link>`, with system-font fallback | Graceful degradation if offline |
| Icons | Hand-authored inline SVGs (no icon library) | Keeps the file dependency-free |

---

## 4. Project structure

Everything lives in **one file**, `AlignLab.html`, organized into four logical sections:

```
AlignLab.html
├── <style>                → CSS variables per theme, layout, component classes
├── <div id="app">         → mount point, fully re-rendered on state change
├── <div id="printable-report"> → hidden node, only populated/shown when printing
└── <script>
    ├── ICONS / icon()     → inline SVG icon set
    ├── baseColor / seqHtml / esc → sequence-letter coloring & HTML-escaping helpers
    ├── needlemanWunsch()  → global alignment (DP + traceback)
    ├── smithWaterman()    → local alignment (DP + traceback)
    ├── PAIRS / PRESETS / RUN_COLUMNS → static reference data
    ├── helixSvg()         → generates the DNA helix SVG from a sequence
    ├── state {}           → single source of truth (theme, page, mode, sequences, scoring…)
    ├── render()           → rebuilds the whole #app innerHTML from state
    ├── renderSidebar / renderMain / renderOverview / renderAnalytics /
    │   renderBenchmarksPage / renderSequencesPage / renderReportsPage /
    │   renderRightPanel   → per-section HTML builders
    ├── buildReportInnerHtml() / printReport() → report generation + print trigger
    └── event listeners    → one delegated 'click' handler, one delegated 'input' handler
```

---

## 5. The algorithms

### Needleman-Wunsch (global alignment)
Builds an `(m+1) × (n+1)` dynamic-programming score matrix `S` where `S[i][j]` is the best score for aligning the first `i` characters of sequence 1 against the first `j` characters of sequence 2.

Recurrence:
```
S[i][j] = max(
  S[i-1][j-1] + (match ? matchScore : mismatchScore),   // diagonal: align s1[i] with s2[j]
  S[i-1][j] + gapScore,                                  // up: gap in sequence 2
  S[i][j-1] + gapScore                                   // left: gap in sequence 1
)
```
First row/column are initialized to `i * gap` / `j * gap` (forcing the whole sequence to be aligned, gaps and all). After filling the matrix, a traceback from `S[m][n]` back to `S[0][0]` reconstructs the actual aligned strings.

- **Time complexity:** `O(m·n)`
- **Space complexity:** `O(m·n)` (could be reduced to `O(min(m,n))` if you only need the score, not the traceback)

### Smith-Waterman (local alignment)
Same recurrence, but with a floor of `0` at every cell (`S[i][j] = max(0, diagonal, up, left)`), and the traceback starts from the single highest-scoring cell in the whole matrix rather than the bottom-right corner, stopping as soon as it hits a `0`. This means the result is the best-scoring *substring* alignment, not the whole sequence — exactly the tool you'd reach for to find a shared motif inside two otherwise-unrelated sequences.

- **Time/space complexity:** same `O(m·n)` as Needleman-Wunsch — only the boundary conditions and traceback termination differ.

### The other three "benchmark" algorithms
NW and SW are the only two actually implemented and run against your live input; **AFF** (affine-gap, Gotoh's algorithm), **BLM** (BLOSUM62-scored alignment), and **MSA** (multiple-sequence, star-alignment consensus) appear in the Benchmarks panel as reference/comparison data to illustrate how alignment strategy choice affects performance and scoring — they're a fixed dataset in the panel rather than something the UI recomputes.

---

## 6. State management (without a framework)

There's a single mutable `state` object:

```js
const state = {
  theme: 'dark', page: 'overview', mode: 'global', modeOpen: false,
  activePair: 'p1', s1: 'GATTACA', s2: 'GCATGCU', editing: false,
  scoring: { m: 2, mm: -1, g: -2, presetId: 'std' },
};
```

Any action mutates `state` directly, then calls `render()`, which regenerates the entire `#app` innerHTML from scratch based on current state — essentially a manual, synchronous version of React's "re-render on state change" model, minus the diffing. This is a deliberate simplicity trade-off: the DOM here is small enough that a full re-render is cheap (sub-millisecond), so there's no need for reconciliation.

**Events are delegated, not rebound.** Rather than attaching a new `click` listener to every button on every render (which would leak listeners or require careful cleanup), there's exactly one `click` listener and one `input` listener on `document`, both added once at load time. Each checks `e.target.closest('[data-action]')` to figure out what was clicked. Because listeners live on `document` rather than on the elements themselves, they keep working after `render()` throws away and rebuilds the DOM underneath them.

**Typing in the sequence inputs is the one place that needs special handling.** A naive "re-render on every keystroke" would blow away and recreate the `<input>` element, which loses focus and cursor position. The fix: on `input`, capture `selectionStart` before re-rendering, update `state`, call `render()`, then re-`focus()` the (newly created) input and restore the cursor position with `setSelectionRange()`. This keeps the alignment live-updating on every keystroke (matching what you'd get "for free" with a controlled React input) without the framework.

---

## 7. Theming

Two palettes (`dark` / `light`) are defined once as CSS custom properties under `:root[data-theme="dark"]` and `:root[data-theme="light"]`. Every component style references `var(--bg)`, `var(--text)`, `var(--blue)`, etc., instead of hardcoded colors. Toggling theme is a single line:

```js
document.documentElement.setAttribute('data-theme', state.theme);
```

The browser repaints every themed element instantly — no re-render of the component tree is required for color changes, only the one line above (though in practice `render()` still runs to update the sun/moon icon and any theme-dependent inline styles, like the benchmark tile tints).

---

## 8. Report generation & printing

The Reports page shows nothing but a button until you click "Print Full Report" — no report is built up front. On click, `buildReportInnerHtml()` assembles a plain-HTML summary (sequences, algorithm, scoring, result, complexity table, benchmark table) as a string, injects it into a `<div id="printable-report">` that's normally `display: none`, and calls `window.print()`.

A `@media print` rule does the swap:
```css
@media print {
  .app { display: none !important; }
  .printable-report { display: block !important; }
}
```
So the interactive app disappears and only the freshly generated report is sent to the printer/PDF — without needing a separate window, a Blob URL, or a server round-trip. This only works cleanly because the app is a real top-level page (opened directly via `file://` or a normal server), rather than embedded inside a sandboxed iframe, which is what an earlier React-based version of this tool ran into (`window.print()`/`window.open()` behave unreliably inside sandboxed preview iframes).

---

## 9. Design decisions worth knowing (interview Q&A style)

**Q: Why vanilla JS instead of React/Vue/etc.?**
The primary requirement was "opens directly in Chrome when clicked" — i.e., no build step, no `node_modules`, works offline, works as a single portable file. React requires either a bundler or a CDN script (which fails offline) plus JSX transpilation. For an app this size (a handful of pages, one shared state object, no deep component trees), a framework's main benefits — diffing, component composition, ecosystem — aren't worth the dependency. The trade-off is that you own re-rendering, event delegation, and focus management yourself, which is exactly why those parts of the code are more deliberate than they'd be in idiomatic React.

**Q: How would this scale if the app grew (more pages, more interactivity)?**
The full-innerHTML-re-render pattern would start to show cracks — bigger DOM trees make full rebuilds more expensive, and things like scroll position, `<video>`/`<canvas>` state, or third-party widget instances don't survive a full re-render the way a text input's value does. At that point I'd either introduce a minimal virtual-DOM diffing step, split `render()` into per-section functions that only re-run when their slice of state changes, or just adopt a framework. The delegated-event-listener pattern, though, scales fine regardless — that part doesn't need to change.

**Q: Why manually escape strings (`esc()`) instead of just using `textContent`?**
Because the render functions build big chunks of markup (icons, layout, styling) as strings and inject them with `innerHTML` for simplicity, any *user-controlled* value — critically, the sequence text a person can type into the edit inputs — has to be HTML-escaped before being interpolated into that markup, or it becomes a straightforward stored/reflected XSS vector (e.g. typing `<img src=x onerror=alert(1)>` as a "sequence"). `esc()` is a small helper that replaces `&`, `<`, `>` before any user input is concatenated into a template string. Everything else (button labels, static data) doesn't strictly need it, but I apply it consistently anywhere user-influenced text is interpolated.

**Q: Walk me through the traceback in Needleman-Wunsch.**
After filling the score matrix forward, I start at the bottom-right cell `S[m][n]` and walk backward. At each cell I re-derive which of the three moves (diagonal, up, left) produced that score and step accordingly: diagonal means "these two characters are aligned" (match or mismatch), up means "insert a gap in sequence 2," left means "insert a gap in sequence 1." I prepend characters to the two output strings as I go, so the alignment is built in reverse and ends up correctly ordered once the walk reaches `(0,0)`.

**Q: What's the actual complexity, and where would this break down at scale?**
Both algorithms are `O(m·n)` time and space. For the sequence lengths used in this app (capped at 20 characters for editing, up to n≤64 in the benchmark comparisons), that's trivial. Real genomic sequences are thousands to billions of bases long, where a naive `O(m·n)` DP matrix becomes infeasible in both time and memory — that's why real-world tools use heuristics (BLAST's seed-and-extend), banded DP (restricting the search to a diagonal band when sequences are expected to be similar), or Hirschberg's algorithm to get the traceback in `O(min(m,n))` space instead of `O(m·n)`.

**Q: How does the "5 vs today's mode" benchmark highlighting work?**
Every column in the Benchmarks panel gets its own accent color and tinted background unconditionally — all five are "highlighted" simultaneously so no algorithm reads as visually deprioritized. A small dot indicator is layered on top of whichever column corresponds to the currently active mode (NW for Global, SW for Local), so you don't lose that context, but it's an addition on top of a uniformly-highlighted set rather than the only column styled at all.

**Q: What would you test if this had a test suite?**
Unit tests on `needlemanWunsch`/`smithWaterman` against known textbook examples (verify score and one valid optimal alignment — note optimal alignments aren't always unique, so tests should assert on score primarily and validate alignment structure rather than exact string equality in ambiguous cases); edge cases (empty sequence, single character, fully identical sequences, completely disjoint sequences); `clean()`'s input sanitization (lowercase input, non-letter characters, length capping); and the print-report HTML builder for correct escaping of malicious input.

**Q: Any known limitations?**
- Sequence length is capped at 20 characters for the interactive editor (a UI/visualization choice, not an algorithmic one — the algorithms themselves handle longer input fine).
- The `esc()`-and-`innerHTML` rendering approach, while covered for the current user-input surface, is inherently more injection-prone than a templating approach that separates markup from data (e.g., DOM APIs or a framework's JSX); it works here because the only user-controlled input is the two sequence strings and both paths that render them use `esc()`.
- No persistence — refreshing the page resets to the default state.
- The three extra benchmark algorithms (AFF, BLM, MSA) are illustrative fixed data, not live computations.

---

## 10. How to run

1. Download `AlignLab.html`.
2. Double-click it, or drag it into a Chrome window, or open it via `File → Open File…`.
3. That's it — no server, no install.

To host it instead of opening it locally, it also works as a static file served from any web server (S3, GitHub Pages, `python -m http.server`, etc.) with no configuration.
