# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A single-file, browser-only sales funnel analysis tool. The entire application is `index.html` — no build step, no dependencies to install, no server. Open it directly in a browser.

## How to develop

**Run locally:** open `index.html` in any browser (`file://` protocol works fine).

**Test with real data:** drag `hubspot-crm-exports-deal-locs-for-commit-2026-04-14.csv` onto the drop zone. The dashboard should render with 5 pipeline tabs (first tab active by default) within ~2 seconds.

**Deploy:** push `index.html` to the `main` branch. GitHub Pages serves it automatically from the repo root.

## Architecture

Everything lives in `index.html` as one `<script>` block. The pipeline is:

```
CSV file → PapaParse → detectSchema() → buildDeals() → compute*() → render*()
```

**Schema detection (`detectSchema`)** — scans CSV headers for the pattern `Date entered "STAGE (PIPELINE)"` using a regex. Builds `stageMap` (key → `{pipeline, stageName, enteredCol, exitedCol}`) and `pipelines` (name → sorted stage array). Stages are sorted by numeric prefix (`1.`, `2.`, …). Won/lost stages detected by `/closed[\s-]*won/i` and `/closed[\s-]*lost/i`. The "store spotlight" stage is the first stage name matching `/store/i`.

**Deal model (`buildDeals`)** — for each row, reads `Date entered` / `Date exited` columns per stage to produce `deal.stages[key] = { entered, exited, days }`. Primary pipeline is inferred from which pipeline has the most stage-entry hits for that deal.

**Compute functions** — pure functions, each takes `(deals, pipelineName, stages, wonNames, lostNames)`:
- `computeFunnel` — returns `{ stages[], totalWon }`. Each stage object has `conversionRate`, `dropRate`, and `currentlyIn` (active deals with no exit date at that stage). Property names must stay in sync with `renderFunnel` — a mismatch silently produces `NaN%`.
- `computeTime` — median + IQR per stage (flags high variance when IQR > 2× median). Also called a second time in `analyzeData` to populate `_stageMedians` for the deal analyzer.
- `computeSignals` — win rate by stage in funnel order; inflection = delta > 0.15; also computes the store card (used as the spotlight when the pipeline owns the store stage)
- `computeStale` — 75th-percentile days-in-stage from closed-lost deals; counts active deals currently over threshold. Also called a second time to populate `_stageThresholds` for the deal analyzer.

**`summaryStats`** — computes the four benchmark cards across ALL deals/pipelines (blended view). Needs `schema.stageMap` (not just `schema.pipelines`) to build `allWonKeys` for the close-date fallback. Signature is `summaryStats(deals, schema)`.

**`summaryStatsForPipeline`** — same shape as `summaryStats` but scoped to a single pipeline. Filters deals to those with at least one stage entry in the pipeline; computes `typicalLength` from first stage entry in that pipeline → close date. Signature is `summaryStatsForPipeline(deals, pl, schema)`. Results cached in `_pipelineStats[pl]` during `analyzeData`; `_pipelineStats['all']` holds the blended result from `summaryStats`.

**Rendering** — `renderFunnel` (CSS bars with live `currentlyIn` count), `renderTimeChart` (Chart.js horizontal bar), `renderSignals` / `renderStale` (HTML tables). Chart instances are stored in `_charts{}` and destroyed before recreation to prevent memory leaks.

**Tab switching** — `switchTab(pl)` toggles `.active` CSS class on tabs and sections, re-renders summary cards via `renderSummaryCards(_pipelineStats[pl], pl)`, and re-renders the spotlight card via `renderStoreCard(_pipelineSpotlight[pl])`. All pipeline sections are pre-rendered on load. `renderSummaryCards` accepts an optional `label` parameter rendered as a `grid-column:1/-1` header inside the summary grid.

**Per-pipeline summary cards** — `_pipelineStats['all']` (blended, via `summaryStats`) is computed for the file info bar only; there is no "All Pipelines" tab. Each `_pipelineStats[pl]` is set from `summaryStatsForPipeline` inside the pipeline loop. Cards and spotlight are rendered after the loop using the first pipeline. When null metrics are encountered, `renderSummaryCards` shows "Not enough data" instead of `—` or zero.

**Per-pipeline spotlight** — `_pipelineSpotlight[pl]` is set to the store card only when `schema.storeStage.pipeline === pl`; otherwise `null` (spotlight hidden). This means switching to a pipeline that contains no store stage hides the spotlight card automatically.

**Stale deal at-risk popup** — `renderStale(thresholds, containerId, pl)` now accepts `pl`. When `t.over > 0`, the cell renders a `.at-risk-btn` button with `data-pl` and `data-stage` attributes. `showAtRiskPopup(btn)` reads those attributes, filters `_allDeals` for active deals currently over the threshold at that stage, and displays a `position:fixed` popup listing deal names sorted alphabetically. The popup closes on any outside click via a `document` click listener. `computeStale` now returns `overDeals` (array of deal objects) in addition to `over` (count) — the threshold-building loop in `analyzeData` only reads `thresh` from the result so this is safe.

**Deal analyzer** — rendered below the pipeline tabs, always visible once a CSV is loaded. Key globals populated in `analyzeData` after the pipeline loop:
- `_stageMedians` — `"stage (pipeline)" → median days` (from `computeTime`)
- `_stageThresholds` — `"stage (pipeline)" → 75th-pct days` (from `computeStale`)
- `_wonNames` / `_lostNames` — `Set<stageName>` copied from `schema.wonNames` / `schema.lostNames`
- `_searchDeals` — full deal list sorted alphabetically, used by the autocomplete

`populateDealSearch(deals)` wires the `#da-input` / `#da-dropdown` autocomplete. `renderDealDetail(deal)` builds the timeline and table for a selected deal. `resetTool()` clears all nine globals (`_charts`, `_allDeals`, `_searchDeals`, `_stageMedians`, `_stageThresholds`, `_wonNames`, `_lostNames`, `_pipelineStats`, `_pipelineSpotlight`) and resets the search input.

**Terminal stage handling** — `isTerminalStage(stageName)` returns true when a stage is in `_wonNames`, `_lostNames`, or matches `/^closed/i`. Terminal stages get special treatment in `renderDealDetail`:
- `days` is set to `null` (no days-from-now calculation); shown as `—`
- `med` and `thresh` are forced to `null` — benchmark columns show `—`
- `status` is `'won'` or `'lost'` instead of the normal pace scale
- Badge renders as `✓ Closed Won` or `✗ Closed Lost` via `STATUS_LABEL`

**Status scale** — `dealStageStatus(days, med, thresh)` returns one of: `ahead | on-pace | behind | stale | no-bench | won | lost`. Thresholds: ahead < 60% of median; on-pace ≤ 140%; stale > threshold; behind is everything else; no-bench when no median data exists. All labels and badge colours are centralised in `STATUS_LABEL` / `STATUS_COLOR` — add new statuses there to keep timeline nodes and table badges in sync.

## Key data contract

The tool auto-detects any HubSpot deals CSV. The only assumption is that HubSpot's standard column naming is used:
- `Deal Stage` — current stage (plain stage name, no pipeline suffix)
- `Date entered "STAGE (PIPELINE)"` — entry timestamp
- `Date exited "STAGE (PIPELINE)"` — exit timestamp
- Dates formatted as `YYYY-MM-DD HH:mm`
- `Close Date` is **optional** — if absent (e.g. live-deals exports), `summaryStats` falls back to the latest `Date entered` among any closed-won stage as a proxy close date

Duration columns (`Latest time in …` / `Cumulative time in …`) are present in the sample CSV but are **not used** — all time calculations derive from `Date entered` / `Date exited` pairs. Median is used throughout; mean is never computed.
