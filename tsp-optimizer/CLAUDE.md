# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

The TSP Optimizer is a planned single-file HTML calculator (`index.html`) for U.S. military members. It is part of the militarywealthcoach.com suite. See the parent `CLAUDE.md` (`../CLAUDE.md`) for all shared conventions: brand/visual system, chart architecture, slider CSS, prefix input fields, JS syntax checking, and Netlify deploy commands. Everything in the parent applies here — do not deviate.

## Tool Purpose

Helps service members decide **how to allocate TSP contributions** — specifically:

- **Traditional vs. Roth TSP** tradeoff over a career timeline (current vs. future marginal tax rates)
- **Contribution percentage vs. take-home pay** impact
- **BRS matching** — for members under the Blended Retirement System, model the DoD 1%–5% match vesting schedule (immediate 1% auto-contribution, matching vests at 2-year mark)
- Optional: fund allocation (C/S/I/F/G funds) with projected growth rates

The primary output is a chart showing projected TSP balance at separation/retirement for Traditional vs. Roth paths, with a crosshair panel showing year-by-year breakdown.

## Architecture

Follows the **SBP Defeater pattern** (see `../CLAUDE.md` → Chart Architecture). Key structure:

- Single global state object (e.g., `G`) holding computed data points and chart reference
- `computePts()` builds the year-by-year dataset; `buildChart()` or `recalc()` renders it
- Local `crosshairPlugin` + `annotPlugin` — never registered globally
- `updateCrosshair(yr, syncSlider)` fully rebuilds `#cpanel` innerHTML each call

## TSP Model Logic

Key parameters and assumptions to encode:

| Parameter | Default / Notes |
|-----------|----------------|
| Contribution rate | User input (1–100% of base pay, capped at IRS limit: $23,500 in 2026) |
| IRS annual limit | $23,500 (2026); model should cap contributions at this ceiling |
| DoD BRS match | 1% auto + up to 4% matching (dollar-for-dollar up to 3%, 50¢/dollar on next 2%) |
| Vesting | BRS match vests at 2-year TIS mark |
| C-Fund growth rate | 10%/yr (≈10.8% since inception; net of fees — same assumption used in BAH tool) |
| Tax assumption | Traditional: contributions deduct now, withdrawals taxed; Roth: contributions after-tax, withdrawals tax-free |
| Compounding | Monthly (`fvPMT(pmt, r/12, yrs*12)` pattern from SBP defeater) |

Use the same `fvPMT(pmt, r, yrs)` helper already established in sibling tools:
```js
function fvPMT(pmt, r, yrs) {
  const m = r/12, n = yrs*12;
  return m === 0 ? pmt*n : pmt*(Math.pow(1+m,n)-1)/m;
}
```

For Traditional vs. Roth comparison, the crosshair panel verdict should flip based on which path yields higher **after-tax** terminal value.

## Input Fields Expected

- Pay grade (E1–E9, W1–W5, O1–O10) — drives base pay lookup
- Years of service at start
- Years until separation/retirement
- Contribution percentage
- Retirement system (Legacy / BRS toggle)
- Current marginal tax rate and expected retirement marginal tax rate (for Traditional vs. Roth verdict)

Base pay table: reuse the `PAY_TABLE` already defined in `../sbp-defeater/index.html` (copy it directly — no shared imports).
