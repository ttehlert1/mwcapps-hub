# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

A suite of single-file HTML financial calculators for U.S. military members, published on militarywealthcoach.com. Each tool lives in its own subdirectory and is one self-contained `index.html` file with no build step, no framework, and no dependencies beyond Chart.js loaded from CDN.

| Tool | Directory | Status |
|------|-----------|--------|
| SBP Defeater | `sbp-defeater/` | Live |
| BAH House-Hack Wealth Builder | `bah-househacker/` | Live |
| Debt & DTI Planner | `debt-dti-planner/` | Planned |
| TSP Optimizer | `tsp-optimizer/` | Planned |
| War Chest Optimizer | `war-chest-optimizer/` | Planned |

## Development Workflow

**Preview locally** — open any `index.html` directly in a browser (no server needed).

**Check JS syntax** (extract script block first):
```bash
sed -n '/<script>/,/<\/script>/{ /<script>/d; /<\/script>/d; p }' index.html | node -e "const s=require('fs').readFileSync('/dev/stdin','utf8'); try{new Function(s);console.log('OK')}catch(e){console.log(e.message)}"
```

**Deploy to Netlify** (requires `netlify login` first, no `netlify.toml` exists — each subdirectory deploys as its own site):
```bash
cd <tool-dir>
npx netlify deploy --prod
```

**Debug JS parse errors** — apostrophes in JS string literals must use `\u2019` (curly apostrophe), not `'`, because strings are single-quoted throughout. Raw apostrophes silently break the parser.

## Brand / Visual System

All tools share the same CSS variables — do not deviate from these:
```css
--dark: #111111    /* backgrounds, slider thumbs, table headers */
--mid:  #8B1515    /* crimson — primary accent, borders, buttons */
--gold: #E8C200    /* hero numbers, header text */
--bg:   #F5F5F5
--card: #FFFFFF
--text: #1A1A1A
--muted: #6B7280
--border: #DEDEDE
--green: #166534
--red:   #991B1B
```
The site header uses `background: var(--mid)` with `color: var(--gold)` text.

## Chart Architecture (SBP Defeater Pattern — reuse in all tools)

All interactive charts follow the pattern established in `sbp-defeater/index.html`. Key conventions:

- **Continuous numeric x-axis** (`type:'linear'`) — never category labels — so `getValueForPixel` / `getPixelForValue` work for fractional positions.
- **`chart._chYr`** — stores the current fractional year; set before every `chart.update('none')`.
- **`crosshairPlugin`** (local, not global) — `afterDatasetsDraw` reads `chart._chYr`, draws a dashed vertical line, then linearly interpolates each dataset's y-value between adjacent labeled points to place colored dots with white stroke rings.
- **`annotPlugin`** (local) — draws gold dashed annotation lines (e.g. PCS year, term expiry) with text labels.
- **`canvasToYear(e)`** — converts mouse/touch clientX to chart year via `chart.scales.x.getValueForPixel`, clamped to axis range.
- **`updateCrosshair(yr, syncSlider)`** — sets `_chYr`, calls `chart.update('none')`, optionally syncs the range input, then rebuilds `#cpanel` innerHTML with `.cp-row1` / `.cp-row2` / `.cp-verdict` structure.
- **`sliderMoved(val)`** — calls `updateCrosshair(parseFloat(val), true)`.
- Mouse events: `onmousemove` → `updateCrosshair(yr, false)` (don't move slider); `ontouchmove` → `updateCrosshair(yr, true)` (sync slider); `onmouseleave` → set `_chYr = undefined`, redraw.
- Fill between lines: `fill: { target: 1, above: 'rgba(22,101,52,0.12)', below: 'rgba(139,21,21,0.10)' }` on dataset 0.

### cpanel HTML structure
```html
<div class="cpanel" id="cpanel">  <!-- fully rebuilt by updateCrosshair -->
  <div class="cp-row1">Year X · context info</div>
  <div class="cp-row2">
    <div><div class="cp-val-label">Label</div><div class="cp-val-num">$XXX</div></div>
    <div class="cp-verdict win|sbp|even">verdict text</div>
    <div style="text-align:right"><div class="cp-val-label">Label</div><div class="cp-val-num">$XXX</div></div>
  </div>
</div>
```

### Slider CSS (use verbatim)
```css
input[type=range] { flex:1; height:6px; padding:0; border:none; border-radius:3px;
  background:var(--border); appearance:none; -webkit-appearance:none; cursor:pointer; }
input[type=range]::-webkit-slider-thumb { -webkit-appearance:none; width:22px; height:22px;
  border-radius:50%; background:var(--dark); border:2.5px solid #fff; box-shadow:0 1px 5px rgba(0,0,0,.3); }
input[type=range]::-moz-range-thumb { width:22px; height:22px; border-radius:50%;
  background:var(--dark); border:2.5px solid #fff; }
```

## BAH House-Hack Data

`bah-househacker/index.html` embeds ~760 KB of DoD 2026 BAH data directly in the JS (no fetch):

- **`ZIP_MHA`** — object mapping 40,959 ZIP codes → MHA code (e.g. `"92101":"CA038"`)
- **`MHA_NAMES`** — object mapping MHA code → display name
- **`BAH_W`** / **`BAH_WO`** — objects mapping MHA code → 27-element rate array (with/without dependents)

**Rate array column index** (0-based): E1–E9 = 0–8, W1–W5 = 9–13, O1E–O3E = 14–16, O1–O10 = 17–26. O-3 with dependents = index 19.

Source files in `bah-househacker/BAH-ASCII-2026/` (not re-embedded unless rates change for a new year).

## BAH Wealth Model Logic

Four housing scenarios compared over 10 years:
1. **On-Base** — $0 wealth; BAH forfeited
2. **Renting** — 2/3 BAH to rent, 1/3 BAH → TSP at 10%/yr compounded monthly
3. **Buy & Live Alone** — VA loan (0% down, 2.15% funding fee financed, 41% DTI); post-PCS: rent = P&I (zero net cash flow); wealth = equity (keep) or invested proceeds (sell)
4. **House Hack** — same out-of-pocket as Buy Alone; roommate income → TSP at 10%/yr

When `numRooms === 0`, the primary strategy switches from House Hack to Buy & Live Alone; the House Hack scenario card is hidden and the hero/chart compare Buy & Live Alone vs the selected alternative.

**Key assumption**: TSP C-Fund return = 10%/yr (≈10.8% since-inception annualized, 10% net of fees).

## SBP Defeater Model Logic

Compares two strategies over a 30-year retirement timeline:
- **SBP** — actuarial present value of surviving spouse annuity stream using `getLE(a) = max(1, 86.04 - 1.3667a + 0.005a²)` (smooth quadratic fit to SSA life tables; do not replace with lookup table — it will produce a step-function curve)
- **Combined (LI + Portfolio)** — term life insurance + self-insured investment portfolio

Key functions: `annuity(first, yrs, cola)` uses closed-form formula (not a loop). Global state in object `G`. Chart initialized at breakeven year (or year 10 if no breakeven).

## Prefix Input Fields

For `$`-prefixed inputs, use this pattern and ensure specificity beats the base `.field input` rule:
```css
.field .prefix-wrap input { padding-left: 52px; }  /* must beat .field input specificity */
.prefix-wrap .prefix { position:absolute; left:11px; top:50%; transform:translateY(-50%); }
```
