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

All tools share one editorial system that matches militarywealthcoach.com.
Cream page, white cards, ONE loud color (crimson). Big numbers are gold serif on
near-black panels. See `BRAND-SPEC.md` for the full component CSS; the shipped tools
(`tsp-optimizer/`, `sbp-defeater/`, `bah-househacker/`) are the spec made real — read
them as reference before building a new tool.

**Fonts** (load in `<head>` before `<style>`):
- Newsreader (serif) — headings + all big numbers, weight 600, letter-spacing -0.01em
- Manrope (sans) — body, inputs, buttons; base 16px / line-height 1.55
- JetBrains Mono — eyebrows, field labels, table headers, tags (always uppercase + tracked)

```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Newsreader:ital,opsz,wght@0,6..72,400;0,6..72,500;0,6..72,600;0,6..72,700;1,6..72,500;1,6..72,600&family=Manrope:wght@400;500;600;700;800&family=JetBrains+Mono:wght@400;500;600;700&display=swap" rel="stylesheet">
```

**Palette** — keep the variable NAMES exactly (scripts inject `var(--muted)` etc. inline);
only these values are the brand. Note `--mid` is the crimson and `--gold` is muted, not neon.

```css
:root {
  --dark:    #1A1A1A;   /* top bar, dark hero/stat panels, table headers, slider context */
  --mid:     #A6182C;   /* crimson — primary accent, buttons, eyebrows, focus ring, borders */
  --gold:    #C9A24B;   /* muted gold — hero/stat numbers ON DARK, accent rules */
  --bg:      #F7F3EB;   /* warm cream — page background, inset boxes */
  --bg-2:    #EFE9DD;   /* deeper cream — neutral tints */
  --card:    #FFFFFF;
  --text:    #1A1A1A;
  --ink-2:   #3D3833;   /* secondary body text */
  --muted:   #6C645B;   /* labels, captions */
  --border:  #E3DCCC;   /* hairline borders */
  --green:   #2F6B43;   /* positive / winning */
  --red:     #7A1020;   /* negative / loss */
  --gold-soft:#E7D9B0;  /* highlighted/recommended table rows, gold tags */
}
```

**Geometry:** `border-radius:2px` everywhere (cards, buttons, inputs, tags). Hairline
`1px solid var(--border)`. Almost no shadow (`0 1px 2px rgba(26,26,26,.03)` on cards).
Insight boxes use `border-left:3px solid var(--gold)` on `--bg`.

**Required chrome on every tool:**
- A dark `.topbar` above the header: "Military Wealth Coach" (links to site root) on the left,
  "← All Tools" (links to `https://militarywealthcoach.com/apps`) on the right — both JetBrains Mono.
- Cream left-aligned header: mono crimson eyebrow with a 24px leading rule → Newsreader serif title → sub.
- Section labels: mono crimson, uppercase, with a trailing hairline rule (`flex:1;height:1px`).
- Primary button: crimson `--mid` fill, white text, NOT uppercase, 2px corners, hover → `--red`.
- Inputs: crimson focus ring `box-shadow:0 0 0 3px rgba(166,24,44,.12)`.
- The headline result lives on a dark panel as a gold Newsreader number.

**Chart.js:** green `#2F6B43` = positive series, crimson `#A6182C` = cost/loss series,
gold `#C9A24B` = annotation/crosshair dashed lines, grid `rgba(26,26,26,0.06)`,
ticks `--muted` in Manrope. Keep the SBP-pattern continuous `type:'linear'` x-axis +
local crosshair/annotation plugins.

**Do NOT** reintroduce `#8B1515`, `#E8C200`, 16px corners, 1.5px borders, or system/Segoe/Roboto
fonts. Never use neon blue/orange/yellow for chart categories — derive a muted slate (`#4A6B86`) instead.

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
- Fill between lines: `fill: { target: 1, above: 'rgba(47,107,67,0.12)', below: 'rgba(166,24,44,0.10)' }` on dataset 0.

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
input[type=range] { flex:1; height:4px; padding:0; border:none; border-radius:2px;
  background:var(--border); appearance:none; -webkit-appearance:none; cursor:pointer; }
input[type=range]::-webkit-slider-thumb { -webkit-appearance:none; width:20px; height:20px;
  border-radius:50%; background:var(--mid); border:3px solid #fff; box-shadow:0 1px 4px rgba(26,26,26,.28); }
input[type=range]::-moz-range-thumb { width:20px; height:20px; border-radius:50%;
  background:var(--mid); border:3px solid #fff; }
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
