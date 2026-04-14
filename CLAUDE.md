# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A single-file, browser-only sales funnel analysis tool. The entire application is `index.html` — no build step, no dependencies to install, no server. Open it directly in a browser.

## How to develop

**Run locally:** open `index.html` in any browser (`file://` protocol works fine).

**Test with real data:** drag `hubspot-crm-exports-deal-locs-for-commit-2026-04-14.csv` onto the drop zone. The dashboard should render across 5 pipeline tabs within ~2 seconds.

**Deploy:** push `index.html` to the `main` branch. GitHub Pages serves it automatically from the repo root.

## Architecture

Everything lives in `index.html` as one `<script>` block. The pipeline is:

```
CSV file → PapaParse → detectSchema() → buildDeals() → compute*() → render*()
```

**Schema detection (`detectSchema`)** — scans CSV headers for the pattern `Date entered "STAGE (PIPELINE)"` using a regex. Builds `stageMap` (key → `{pipeline, stageName, enteredCol, exitedCol}`) and `pipelines` (name → sorted stage array). Stages are sorted by numeric prefix (`1.`, `2.`, …). Won/lost stages detected by `/closed[\s-]*won/i` and `/closed[\s-]*lost/i`. The "store spotlight" stage is the first stage name matching `/store/i`.

**Deal model (`buildDeals`)** — for each row, reads `Date entered` / `Date exited` columns per stage to produce `deal.stages[key] = { entered, exited, days }`. Primary pipeline is inferred from which pipeline has the most stage-entry hits for that deal.

**Compute functions** — pure functions, each takes `(deals, pipelineName, stages, wonNames, lostNames)`:
- `computeFunnel` — returns `{ stages[], totalWon }`. Each stage object uses the property name `conversionRate` (not `convRate`) — these must stay in sync with `renderFunnel` which reads `f.conversionRate` and `f.dropRate`.
- `computeTime` — median + IQR per stage (flags high variance when IQR > 2× median)
- `computeSignals` — win rate by stage in funnel order; inflection = delta > 0.15; also computes the cross-pipeline store card
- `computeStale` — 75th-percentile days-in-stage from closed-lost deals; counts active deals currently over threshold

**`summaryStats`** — computes the four benchmark cards. Needs `schema.stageMap` (not just `schema.pipelines`) to build `allWonKeys` for the close-date fallback. Signature is `summaryStats(deals, schema)`.

**Rendering** — `renderFunnel` (CSS bars), `renderTimeChart` (Chart.js horizontal bar), `renderSignals` / `renderStale` (HTML tables). Chart instances are stored in `_charts{}` and destroyed before recreation to prevent memory leaks. Tab switching just toggles `.active` CSS class — all pipeline sections are pre-rendered on load.

## Key data contract

The tool auto-detects any HubSpot deals CSV. The only assumption is that HubSpot's standard column naming is used:
- `Deal Stage` — current stage (plain stage name, no pipeline suffix)
- `Date entered "STAGE (PIPELINE)"` — entry timestamp
- `Date exited "STAGE (PIPELINE)"` — exit timestamp
- Dates formatted as `YYYY-MM-DD HH:mm`
- `Close Date` is **optional** — if absent (e.g. live-deals exports), `summaryStats` falls back to the latest `Date entered` among any closed-won stage as a proxy close date

Duration columns (`Latest time in …` / `Cumulative time in …`) are present in the sample CSV but are **not used** — all time calculations derive from `Date entered` / `Date exited` pairs. Median is used throughout; mean is never computed.
