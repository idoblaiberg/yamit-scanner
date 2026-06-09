# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the app locally

```bash
python3 -m http.server 8080
# open http://localhost:8080
```

Camera APIs require HTTPS or `localhost` — never `file://`. There is no build step, no package manager, no tests.

## Architecture

The entire app is `index.html` — one file, inline CSS and JS, no framework. Deployable directly to GitHub Pages. This is a hard constraint: do not introduce a build step, bundler, or npm dependencies.

**CDN-only libraries:**
- `@zxing/library@0.19.1` — barcode decoding fallback (global: `ZXing`)
- `PapaParse 5.4.1` — CSV parsing (global: `Papa`)

**Barcode detection priority:** Native `BarcodeDetector` API first (Chrome 83+ / Android 9+), ZXing as fallback. Always use `facingMode: { ideal: 'environment' }` for back camera.

## Data model

Two independent CSV sources, both cached to `localStorage` as raw text and re-parsed on boot:

| Source | Global | localStorage key | Column map key |
|---|---|---|---|
| Inventory | `products[]` | `yamit_data` | `cols` (`yamit_cols`) |
| Scraping/price report | `scrapeMap{}` | `yamit_scrape_data` | `scrapeCols` (`yamit_scrape_cols`) |

`scrapeMap` is an O(1) lookup object `{ [sku]: row }` built by `buildScrapeMap()`. It is joined onto inventory rows at render time in `makeCard()` and `openResult()` — not merged into `products[]`.

`pendingBarcodes[]` — barcodes scanned but not found in inventory — is persisted to `yamit_pending`.

## Key functions

| Function | What it does |
|---|---|
| `parseCSV(text)` | PapaParse wrapper; filters empty rows and `***…***` category headers; uses `cols` mapping |
| `buildScrapeMap(text)` | Parses price report; builds `scrapeMap` keyed by `scrapeCols.sku` |
| `doSearch(q)` | Case-insensitive partial match across `cols.id`, `cols.name`, `cols.alt` |
| `handleBarcode(val)` | Exact-match on id/alt/ean → fallback to `doSearch` → opens result if unique |
| `openResult(p)` | Opens bottom-sheet modal; appends price section from `scrapeMap` if available |
| `makeCard(p, onClick)` | Builds product list-card DOM node; inlines price badge from `scrapeMap` |
| `savePendingBarcode()` | Pushes current `pendingScan.barcode` to `pendingBarcodes[]`, deduplicates, persists |
| `linkPendingToProduct(sku)` | Sets `resolvedSku` on the most recent unresolved pending entry |
| `exportPendingCSV()` | Downloads `pendingBarcodes[]` as CSV; prompts to clear list |

## UI / DOM conventions

- All DOM construction uses `textContent`, `createElement`, `setAttribute` — never `innerHTML`
- RTL: `<html dir="rtl" lang="he">`, search inputs use `direction: rtl`
- Modals use `.open` class toggle (CSS `display: none` → `display: flex`)
- ES5 (`var`, `function`) throughout — required for compatibility with older Android WebView

## Screens & overlays (current, being redesigned)

```
setupScreen        — first-launch data load
mainScreen         — search bar + results list
  camOverlay       — full-screen camera (fixed, z-index 300)
    scanPreview    — bottom sheet after capture (z-index 200)
  resultModal      — product detail bottom sheet (z-index 200)
  settingsModal    — settings bottom sheet (z-index 200)
```

## Pending UX redesign (in progress)

The app is being redesigned to be scanner-first: camera opens immediately on launch, continuous auto-scan (rAF BarcodeDetector loop), result overlay on frozen frame, direct product detail or link-to-product flow. The current tap-to-capture + scan-preview sheet pattern is being replaced.
