# Changelog

All notable changes to this project are documented here.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

---

## [1.1.0] — 2026-05-13

### Added
- **Price enrichment from scraping report** — a second data source (daily Google Sheets CSV) can now be loaded alongside the inventory CSV
- Product cards now display sale price (green) and regular price (struck-through) when a sale exists
- Result modal now shows a "נתוני אתר" section with: regular price, sale price, online stock availability, two feature fields, and a clickable product URL
- Settings modal: new "נתוני מחירים (סקרייפינג)" section with URL field, load button, and green item count indicator
- Settings modal: new column-mapping section for all 7 scraping report fields (`מק"ט`, `מחיר רגיל (₪)`, `מחיר מבצע (₪)`, `מלאי`, `תכונה 1`, `תכונה 2`, `URL מוצר`)
- Three new `localStorage` keys: `yamit_scrape_url`, `yamit_scrape_data`, `yamit_scrape_cols`
- Scraping data persists across sessions; restored from `localStorage` on boot
- "Clear data" now wipes both inventory and scraping data

### Technical
- `buildScrapeMap(csvText)` — parses scraping CSV into an O(1) lookup map keyed by SKU
- `loadScrapeUrl(url)` — fetch → parse → persist → toast pattern, mirrors existing `loadUrl()`
- `updateScrapeCount()` — keeps the Settings item count indicator in sync

---

## [1.0.0] — 2026-05-12

### Added
- Initial release
- Setup screen: load inventory from Google Sheets CSV URL or local file upload
- Main screen: text search (name, SKU, alternative number) with debounce
- Color-coded stock badges: green (positive) / gray (zero) / red (negative/deficit)
- Barcode scanner via ZXing — exact match on SKU and alternative number, fallback to partial text search
- Result modal: item number, alternative number, stock quantity
- Settings modal: CSV URL, file re-upload, column mapping for all four inventory fields
- All data persisted in `localStorage` — works offline after first load
- Hebrew / RTL throughout, Heebo font, mobile-first layout
- GitHub Pages deployment
