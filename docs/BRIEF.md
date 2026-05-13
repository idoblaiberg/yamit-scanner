# Claude Code Brief — Yamit Scanner App

## What to build

A single `index.html` file — a barcode scanner + inventory lookup tool for Yamit surf store staff.
No backend, no build step, no frameworks. Pure HTML/CSS/JS that GitHub Pages can serve directly.

---

## Deployment target

**GitHub Pages** — the file must be named `index.html` and live in the repo root.
This matters because camera access (`getUserMedia`) is blocked on `file://` — HTTPS is required.
Live URL will be: `https://[username].github.io/yamit-scanner/`

---

## Stack

| Concern | Solution |
|---|---|
| Barcode scanning | ZXing-js (CDN) — `@zxing/library` |
| CSV parsing | PapaParse (CDN) |
| UI language | Hebrew, RTL (`dir="rtl"`) |
| Font | Heebo (Google Fonts) |
| Persistence | `localStorage` only |
| Backend | None |

CDN links to use:
```html
<script src="https://unpkg.com/@zxing/library@latest/umd/index.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/PapaParse/5.4.1/papaparse.min.js"></script>
<link href="https://fonts.googleapis.com/css2?family=Heebo:wght@400;500;700&display=swap" rel="stylesheet">
```

---

## Data sources

### Source 1 — Inventory CSV (warehouse stock)

The store exports a CSV with these exact column headers:

```
מ.במחסן,מחס,מספר פריט,שם פריט,מספר חליפי,***,כמות במלאי
```

| Column | Description |
|---|---|
| `מ.במחסן` | Warehouse flag |
| `מחס` | Warehouse number |
| `מספר פריט` | **Primary ID / barcode match** |
| `שם פריט` | Item name |
| `מספר חליפי` | Alternative / manufacturer number (secondary barcode match) |
| `***` | Unknown flag — ignore |
| `כמות במלאי` | Stock quantity — can be negative (deficit) |

### Source 2 — Scraping Report (prices & features)

A separate Google Sheet (published as CSV) refreshed every morning with scraped website data. Joined to the inventory CSV on `מק"ט` ↔ `מספר פריט`.

```
ענף ספורט,קטגוריה,שם מוצר,תכונה 1,תכונה 2,מק"ט,מחיר רגיל (₪),מחיר מבצע (₪),מלאי,URL מוצר
```

| Column | Description |
|---|---|
| `מק"ט` | **Join key** — matched against `מספר פריט` from the inventory CSV |
| `מחיר רגיל (₪)` | Regular price |
| `מחיר מבצע (₪)` | Sale price (may be empty or equal to regular price) |
| `מלאי` | Online stock / availability status |
| `תכונה 1` | Product feature / spec 1 |
| `תכונה 2` | Product feature / spec 2 |
| `URL מוצר` | Direct link to product page |
| `ענף ספורט`, `קטגוריה`, `שם מוצר` | Metadata — displayed only in the modal |

All column names are configurable in Settings → "מיפוי עמודות – דוח מחירים".

### Filtering rules

- Filter out category header rows. These are rows where `שם פריט` looks like `******* סירות לייזר **********` (starts and ends with asterisks/stars).
- Filter out rows where `שם פריט` is empty AND `מספר פריט` is empty.

### Sample rows

```
,0,5,שובר מתנה (כמות במינוס) 50שנה,,,2.00
,0,50,"משלוח י.ר עד 20 ק""ג ..........",,,-5.00
,0,1,******* סירות לייזר **********,,,
,0,1000000000,סירת לייזר E6 RADIAL ILCA,E6IBFR4,,
9841013.00,0,1006,סירת לייזר PICO  זוגית,14000,,
,0,111608,רצועת איזון מרופדת לייזר ILCA,E6ID02,,-2.00
```

---

## Features

### Screen 1 — Setup (shown on first load, no data yet)
- Title: "סורק מלאי יאמיט"
- Two ways to load data:
  1. **Google Sheets URL** — user pastes a "Publish to web → CSV" link. Fetch it, parse it with PapaParse, store raw CSV text in `localStorage` under key `yamit_url` (store the URL) and `yamit_data` (store the raw CSV).
  2. **File upload** — `<input type="file" accept=".csv">`. Read with FileReader, parse, store in `localStorage` as `yamit_data`.
- After loading, auto-advance to Search screen.
- Show item count on success.

### Screen 2 — Search / Main screen
- **Search bar** — text input. Searches `מספר פריט`, `שם פריט`, `מספר חליפי` (case-insensitive, partial match).
- **Scan button** (📷) — opens camera overlay.
- **Stats bar** — "X פריטים | Y במחסן" or similar.
- **Results list** — show matching items as cards. Each card shows:
  - שם פריט (name) — large
  - מספר פריט (SKU) — small
  - כמות במלאי (stock) — colored badge:
    - **Green** — positive stock
    - **Gray** — zero stock
    - **Red** — negative stock (deficit)
  - Price from scraping report (if loaded):
    - Sale price in green + struck-through regular price when a sale exists
    - Regular price only when no sale
- Clicking a card opens the Result Modal.
- **Settings button** (⚙️) in header — opens Settings Modal.

### Camera overlay
- Full-screen dark overlay with `<video>` element for live camera feed.
- Scan frame/reticle in the center.
- ZXing continuously scans for barcodes.
- On successful scan: close camera, search for scanned value in `מספר פריט` and `מספר חליפי`. If found, show Result Modal. If not found, show brief "לא נמצא" toast.
- Close button (✕) always visible.

### Result Modal
- Bottom sheet / card style.
- **Inventory section:** שם פריט, מספר פריט, מספר חליפי (if exists), כמות במלאי (colored badge).
- **"נתוני אתר" section** (shown only when scraping report is loaded and SKU matches):
  - מחיר רגיל, מחיר מבצע (if different from regular)
  - מלאי באתר (online availability)
  - תכונה 1, תכונה 2 (if non-empty)
  - "פתח באתר ↗" — clickable link to product page (opens in new tab)
- Close button.

### Settings Modal
- **מקור נתונים** — inventory CSV URL (prepopulated from `localStorage`). Save & reload button. Re-upload file button.
- **מיפוי עמודות** — 4 text inputs for inventory column names (`id`, `name`, `alt`, `stock`), saved to `localStorage('yamit_cols')` as JSON. Defaults: `{id: "מספר פריט", name: "שם פריט", alt: "מספר חליפי", stock: "כמות במלאי"}`.
- **נתוני מחירים (סקרייפינג)** — URL field for the scraping report Google Sheets CSV. "שמור וטען" button calls `loadScrapeUrl()`. Shows a green count of matched items once loaded.
- **מיפוי עמודות – דוח מחירים** — 7 text inputs for scraping report column names, saved to `localStorage('yamit_scrape_cols')` as JSON. Defaults match the report's exact headers.
- **Clear data** button — clears all localStorage keys (inventory + scraping), resets state, goes back to Setup screen.

---

## localStorage keys

| Key | Value |
|---|---|
| `yamit_data` | Raw inventory CSV text |
| `yamit_url` | Inventory Google Sheets CSV URL |
| `yamit_cols` | JSON: `{id, name, alt, stock}` column name overrides |
| `yamit_scrape_data` | Raw scraping report CSV text |
| `yamit_scrape_url` | Scraping report Google Sheets CSV URL |
| `yamit_scrape_cols` | JSON: `{sku, price, salePrice, stock, feature1, feature2, url}` column name overrides |

---

## UI / Design requirements

- **RTL** throughout — `<html dir="rtl" lang="he">`
- Font: Heebo (Hebrew-optimized)
- Mobile-first — designed for iPhone use in-store
- Dark-ish header, clean white cards
- Stock badge: green (`#22c55e`) / gray (`#9ca3af`) / red (`#ef4444`)
- Camera overlay: full screen, dark background (`rgba(0,0,0,0.9)`)
- Smooth transitions where possible (not required if complicating)

---

## Error handling

- If Google Sheets URL fetch fails (CORS, network): show Hebrew error "שגיאה בטעינת הנתונים. נסה שוב."
- If camera access denied: show message "אין גישה למצלמה. אנא אשר הרשאה בהגדרות."
- If barcode not found in data: show toast "לא נמצא: [barcode value]"
- If CSV has no rows after filtering: show "לא נמצאו פריטים בקובץ"

---

## File structure

Single file only:
```
index.html   ← everything: HTML + embedded <style> + embedded <script>
```

No external files, no build step. The entire app is self-contained in one HTML file.

---

## What NOT to include

- No login / auth
- No write-back to CSV
- No print functionality
- No multi-user / sync
- No service worker / PWA (nice-to-have but skip for v1)

---

## Acceptance criteria

1. Open `index.html` served from GitHub Pages on an iPhone.
2. Load a CSV (either via URL or file upload). See item count.
3. Type a partial item name → see filtered results with colored stock badges.
4. Tap 📷 → camera opens → point at a barcode → item result appears (or "not found" toast).
5. Open Settings → change CSV URL → reload data → item count updates.
6. Open Settings → paste scraping report CSV URL → tap "שמור וטען" → green item count appears.
7. Search for a product that exists in both files → card shows price; tap it → modal shows "נתוני אתר" section with price, features, and clickable URL.
8. Products with no scraping match show no price / no "נתוני אתר" section — no errors.
9. All text is in Hebrew, layout is RTL, font is Heebo.
10. App works fully offline after initial load (both data sets stored in localStorage).

---

## Notes for Claude Code

- Test the CSV parser logic in isolation first — write a small Node.js snippet that parses the sample rows above and prints filtered results. Verify category headers are excluded.
- The ZXing CDN UMD build exposes `ZXing` as a global. Use `ZXing.BrowserMultiFormatReader`.
- PapaParse CDN exposes `Papa` as a global. Use `Papa.parse(csvText, {header: true, skipEmptyLines: true})`.
- Column names in the CSV contain Hebrew characters — make sure string comparisons use the exact same encoding (UTF-8).
- Google Sheets "Publish to web → CSV" URLs do support CORS from browsers. Test with a real URL if possible.
- The app should work on iOS Safari — avoid APIs with poor iOS support. `getUserMedia` with `{video: {facingMode: "environment"}}` works on iOS 14.3+.
- ZXing on iOS: use `codeReader.decodeFromVideoDevice(null, videoElement, callback)` — passing `null` for deviceId lets the browser pick the rear camera (or ZXing will try each available device).
