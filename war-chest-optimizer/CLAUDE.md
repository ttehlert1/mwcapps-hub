# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

War Chest Optimizer is a planned single-file HTML financial calculator for U.S. military members, part of the MWC app suite. It will live at `war-chest-optimizer/index.html` with no build step, no framework, and no external dependencies beyond Chart.js from CDN.

**Suite-wide conventions** (brand colors, chart architecture, crosshair pattern, slider CSS, prefix inputs, deploy workflow) are documented in `../CLAUDE.md` — read that first before building or modifying anything here.

## Planned Scope

This tool is not yet implemented. When built, it should help military members optimize their liquid emergency fund ("war chest") — modeling the tradeoff between cash savings earning HYSA/money-market yields vs. paying down high-interest debt vs. investing in TSP/brokerage accounts, factoring in military-specific considerations (deployment savings, SCRA 6% interest cap, SDP, etc.).

## Build Checklist (when implementing)

- One `index.html`, self-contained, no external files except Chart.js CDN
- Follow the crosshair/cpanel/slider pattern from `sbp-defeater/index.html` exactly
- Use the brand CSS variables from `../CLAUDE.md` — no deviations
- Test apostrophes in JS strings: use `’`, never a raw `'` inside single-quoted strings
- Validate JS syntax before considering done:
  ```bash
  sed -n '/<script>/,/<\/script>/{ /<script>/d; /<\/script>/d; p }' index.html | node -e "const s=require('fs').readFileSync('/dev/stdin','utf8'); try{new Function(s);console.log('OK')}catch(e){console.log(e.message)}"
  ```
