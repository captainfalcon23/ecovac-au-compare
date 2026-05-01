# DEEBOT AU Comparison Widget

A self-contained, single-file HTML comparison tool for the current Ecovacs DEEBOT robot vacuum lineup sold in Australia. No build step, no dependencies, no server — open the file in a browser and it works.

<img width="3799" height="1840" alt="image" src="https://github.com/user-attachments/assets/cf601953-3448-48b4-ab4e-46f56febb206" />

---

## Features

- **11 models** covering the X11, X8, T90, T80, T50, and T30S families (T50 MAX PRO OMNI excluded — discontinued AU)
- **Three views** — Grid (card overview), List (full sortable table), Compare (side-by-side spec table)
- **Tier-coloured fields** — Navigation type, mop system, and filter level use colour-coded badges (gold → green → blue → neutral → red) to indicate best-to-worst within category
- **Multi-column sort** in list view — click to sort, Shift+click to add secondary sorts with priority badges
- **Unlimited model selection** for comparison — no cap, select as many as you want
- **Per-retailer pricing** — Ecovacs.com/au, JB Hi-Fi, Harvey Norman, and Good Guys where stocked, with best-price callout
- **Key feature data** from Ecovacs marketing docs, filling gaps the spec table misses: hot air dry temperature (63°C vs 45°C matters), AIVI version, ZeroTangle version, TruEdge version, Triple Lift, Fresh-Flow, auto re-mop, bagless status, PowerBoost flash-charge rate, carpet care mode, camera resolution
- **Provenance notes** — X8 PRO OMNI flagged as US-spec unit with no official AU listing

---

## Usage

Download `deebot_compare.html` and open it in any modern browser. No internet connection required after download (fonts load from Google Fonts, everything else is inline).

---

## Data sources

| Data type | Source |
|---|---|
| Spec tables | Extracted from ecovacs.com/au via Playwright (ecovacs.com/us for X8 PRO OMNI) |
| Key features | Ecovacs product page marketing copy |
| Prices | Manually verified across Ecovacs AU, JB Hi-Fi, Harvey Norman, Good Guys — May 2025 |
| Release years | Approximate, based on AU availability |

Prices reflect current discounts at time of last update and will drift. The widget is a point-in-time snapshot, not a live feed.

---

## Adding a new model

See prompt-to-update-html.md for a full prompt you can give to Claude or another AI to add a new model. It covers:

- How to extract the spec table (Playwright, Chrome extension JS injection, or screenshot fallback)
- Which constants to update in the script block
- How to populate the `FEATURES` entry for marketing data not in the spec table
- Checklist before finishing

The key things you'll need to provide:
1. The Ecovacs spec page URL (ending in `#specification`)
2. Retailer URLs and current prices for whichever of Ecovacs / JB / Harvey Norman / Good Guys stock it
3. The product page key features blurb if you have it (saves the AI guessing AIVI version, hot dry temp, etc.)

---

## Known caveats

**X8 PRO OMNI** — Ecovacs AU has no official listing for this model. JB Hi-Fi and The Good Guys appear to be selling the US-spec unit (110V heating module). Specs are sourced from ecovacs.com/us. Confirm Australian warranty coverage before purchasing.

**T50 MAX PRO OMNI** — Included in the raw spec data but excluded from the widget as it appears discontinued in Australia.

**Prices** — These are checked manually and will go stale. Always verify before purchasing.

---

## File structure

Everything lives in a single file — no external JS, no CSS files, no build tooling.

```
deebot_compare.html
├── <style>          CSS (custom properties, grid, list table, compare table)
├── HTML             Header, tab bar, controls, sort bar, notice bar, view containers
└── <script>
    ├── PRICES       Per-retailer pricing by model
    ├── RETAILER_URLS
    ├── SPEC_URLS
    ├── RELEASE_YEARS
    ├── FEATURES     Supplementary data from marketing docs
    ├── RAW          Raw spec table key-value pairs from Playwright extraction
    ├── normalize()  Converts RAW + FEATURES → unified model object
    ├── MODELS       Filtered, normalised array (T50 MAX PRO OMNI excluded)
    ├── renderGrid()
    ├── renderList()
    └── renderCompare()
```

---

## Licence

Do whatever you want with it.
