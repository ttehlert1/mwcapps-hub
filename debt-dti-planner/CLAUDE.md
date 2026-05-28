# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Context

This is the **Debt & DTI Planner** tool, one calculator in the MilitaryWealthCoach suite. All shared conventions — brand colors, chart architecture, slider CSS, JS syntax rules, deployment workflow — live in the parent `../CLAUDE.md`. Read that file first.

This tool is a single self-contained `index.html` (no build step, no framework, Chart.js from CDN).

## Tool Purpose

Helps U.S. military members model debt payoff strategies and understand how debt load affects VA loan eligibility via Debt-to-Income (DTI) ratio.

Expected inputs:
- Gross monthly income (base pay + BAH + BAS, or custom)
- List of debts: balance, interest rate, minimum payment
- Extra monthly payment available to apply

Expected outputs:
- Payoff timeline comparison: **Avalanche** (highest-rate first) vs **Snowball** (smallest-balance first)
- Total interest paid under each strategy
- Monthly DTI over time (front-end and back-end ratios)
- VA loan eligibility threshold marker at 41% DTI

## Key Military-Specific Constraints

- VA loan qualifying DTI ceiling is **41%** (back-end); flag when the user is above/below this line.
- Income inputs should default to tax-free allowances (BAH, BAS) being included in gross for DTI purposes — this matches VA underwriting.
- BAH varies by duty station and dependency status; consider linking to or reusing the MHA lookup already built in `../bah-househacker/index.html`.

## Chart Notes

Follow the SBP Defeater crosshair/cpanel pattern from `../CLAUDE.md`. The x-axis should be months (or years) to payoff. Annotate the month when DTI drops below 41% with a gold dashed `annotPlugin` line.
