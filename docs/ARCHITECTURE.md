# Architecture

## Overview

The entire application is a single `index.html` file containing embedded CSS and JavaScript. There is no build step, no framework, no backend.

```
Browser
  └── index.html
        ├── <style>   — all CSS
        ├── <script>  — all JS
        └── CDN deps  — ZXing (barcode), PapaParse (CSV)
```

This is an intentional constraint: the app must be deployable to GitHub Pages as-is and work on an iPhone Safari browser in a retail environment.

---

## Data Flow

```
[Google Sheets CSV URL]          [Local file upload]
         │                                │
         └──────────┬─────────────────────┘
                    ▼
             fetch / FileReader
                    │
                    ▼
           PapaParse (header:true)
                    │
                    ▼
         Filter invalid rows          ← category headers (***…***), empty rows
                    │
                    ▼
         products[] array             ← in-memory, rebuilt on each load
                    │
         localStorage (raw CSV text) ← persisted for offline / page reload
```

The scraping report follows the same path but builds a `scrapeMap{}` lookup object:

```
[Scraping report CSV]
         │
    PapaParse
         │
         ▼
   scrapeMap{}   { [sku]: row }   ← O(1) lookup at render time
```

At render time, `scrapeMap[product.sku]` is accessed in `makeCard()` and `openResult()` to attach price/feature data.

---

## State

All state is global module-level variables (no framework needed at this scale):

| Variable | Type | Description |
|---|---|---|
| `products` | `Array<Object>` | Parsed inventory rows |
| `cols` | `Object` | Column name mapping for inventory CSV |
| `scrapeMap` | `Object` | SKU → scraping row lookup map |
| `scrapeCols` | `Object` | Column name mapping for scraping CSV |

**Persistence:** on every successful load, raw CSV text is written to `localStorage`. On boot, it is restored and re-parsed — so the app works offline after the first load.

---

## Key Functions

| Function | Purpose |
|---|---|
| `parseCSV(text)` | PapaParse wrapper; filters invalid rows; uses `cols` mapping |
| `buildScrapeMap(text)` | Parses scraping CSV; builds `scrapeMap` keyed by `scrapeCols.sku` |
| `loadUrl(url)` | Fetch → parse → persist inventory; shows toast |
| `loadScrapeUrl(url)` | Fetch → parse → persist scraping report; shows toast |
| `doSearch(q)` | Filters `products` by name, id, alt — case-insensitive partial match |
| `makeCard(p, onClick)` | Builds product card DOM node; inlines price from `scrapeMap` |
| `openResult(p)` | Populates and opens result bottom sheet; appends "נתוני אתר" section if scraping data exists |
| `handleBarcode(val)` | Exact-match SKU/alt lookup; falls back to `doSearch`; opens result if unique |

---

## Libraries

### ZXing `@0.19.1`
- `ZXing.BrowserMultiFormatReader` — continuous decode from `<video>` element
- Loaded from unpkg CDN; exposed as `ZXing` global
- `decodeFromVideoDevice(null, videoEl, callback)` — `null` deviceId lets the browser pick the rear camera (works on iOS 14.3+)

### PapaParse `5.4.1`
- `Papa.parse(text, { header: true, skipEmptyLines: true })` — returns `{ data: Array<Object> }`
- Column names become object keys; handles quoted fields and Hebrew UTF-8 correctly

---

## Security Notes

- No user-supplied HTML is ever injected — all DOM construction uses `textContent`, `createElement`, and `setAttribute`
- External links from the scraping report use `rel="noopener noreferrer"`
- CSV data may contain sensitive pricing — see `.gitignore` for exclusion rules

---

## localStorage Keys

| Key | Value |
|---|---|
| `yamit_data` | Raw inventory CSV text |
| `yamit_url` | Inventory Google Sheets CSV URL |
| `yamit_cols` | JSON: `{id, name, alt, stock}` |
| `yamit_scrape_data` | Raw scraping report CSV text |
| `yamit_scrape_url` | Scraping report Google Sheets CSV URL |
| `yamit_scrape_cols` | JSON: `{sku, price, salePrice, stock, feature1, feature2, url}` |
