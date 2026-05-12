# יאמיט סורק מלאי — Yamit Inventory Scanner

A mobile-first, single-file inventory scanner built for warehouse use.  
Scan barcodes with the device camera, or type/search by name or SKU — results come from a Google Sheets CSV or a local file loaded once and cached on the device.

---

## Features

| Feature | Details |
|---|---|
| **Barcode scanning** | Native `BarcodeDetector` API (Android Chrome 83+) with ZXing fallback |
| **Back-camera auto-select** | Always prefers the environment-facing camera |
| **CSV data source** | Google Sheets published-as-CSV URL, or upload a local `.csv` file |
| **Offline after first load** | Data cached in `localStorage` — no network needed on subsequent visits |
| **Live search** | Searches name, primary SKU, and alternate barcode simultaneously |
| **Column mapping** | Configurable from Settings — works with any Hebrew or English column names |
| **Stock badges** | Color-coded: green (in stock), grey (zero), red (deficit) |
| **RTL Hebrew UI** | Heebo font, fully right-to-left |
| **No build step** | Single `index.html` — open in browser or host anywhere |

---

## Quick Start

### Option A — Google Sheets (recommended for teams)

1. Open your Google Sheet with the inventory data.
2. **File → Share → Publish to web** → choose **CSV** → click **Publish**.
3. Copy the URL (looks like `https://docs.google.com/spreadsheets/d/.../pub?output=csv`).
4. Open the app and paste the URL into the setup field.

The app re-fetches data every time you reload the page. To force a refresh, open **Settings → שמור וטען מחדש**.

### Option B — Local CSV file

Tap **העלה קובץ CSV מהמכשיר** on the setup screen and pick a `.csv` file from your device.

---

## Expected CSV Format

The app works with any CSV that has these four columns (names are configurable in Settings):

| Column (default name) | Purpose |
|---|---|
| `מספר פריט` | Primary item ID / barcode |
| `שם פריט` | Item name shown in search results |
| `מספר חליפי` | Alternate barcode (manufacturer barcode, EAN, etc.) |
| `כמות במלאי` | Stock quantity (numeric) |

Example:

```csv
מספר פריט,שם פריט,מספר חליפי,כמות במלאי
12345,כיסא משרדי,7290012345678,14
98765,שולחן עבודה,,0
11111,מדף מתכת,7290011111111,-3
```

Rows where both name and ID are empty are ignored. Rows matching the pattern `***..***` (section headers) are filtered out automatically.

---

## Column Mapping

If your spreadsheet uses different column names, go to **⚙️ הגדרות → מיפוי עמודות** and update the four field names to match your CSV headers exactly (case-sensitive).

---

## Barcode Scanning

Tap the green camera button next to the search box.

**How detection works:**

1. **Native `BarcodeDetector`** (Chrome 83+ on Android, Chrome 88+ on desktop) — fastest, most reliable. Runs a `requestAnimationFrame` decode loop directly on the video feed. Supports EAN-13, EAN-8, Code 128, Code 39, QR, UPC-A/E, ITF, PDF417, Data Matrix, Aztec.

2. **ZXing fallback** (`@zxing/library@0.19.1`) — used on browsers without `BarcodeDetector`. Enumerates camera devices and selects the back camera by label before starting the decode loop.

On first camera use the browser will ask for permission — tap **Allow**.

**Matching logic after a scan:**

- Exact match on `מספר פריט` → open result immediately
- Exact match on `מספר חליפי` → open result immediately
- Partial text search → show list (auto-open if only one result)
- No match → toast notification

---

## Deployment

The app is a single static file with no server-side requirements.

```
# Serve locally for development
npx serve .
# or
python3 -m http.server 8080
```

For production, drop `index.html` onto any static host:

- **GitHub Pages** — push to `main`, enable Pages from Settings
- **Netlify / Vercel** — drag and drop the file
- **Any web server** — copy the file to the document root

> **HTTPS required for camera access.** `localhost` works without it; any other host must use HTTPS.

---

## Architecture

```
index.html
├── <style>        inline CSS, mobile-first, RTL
├── HTML           setup screen, main screen, camera overlay, modals
└── <script>       vanilla JS, no framework
    ├── boot()          restore saved data from localStorage on page load
    ├── loadUrl()       fetch CSV from Google Sheets URL
    ├── loadFile()      read CSV from FileReader
    ├── parseCSV()      PapaParse wrapper with column filtering
    ├── doSearch()      fuzzy substring search across three columns
    ├── renderList()    DOM card builder for results
    ├── openResult()    product detail bottom sheet
    ├── openCam()       scanner entry point
    │   ├── startNativeScan()   BarcodeDetector + rAF loop
    │   └── startZXingScan()    ZXing device enumeration + decode loop
    ├── onBarcodeFound()        flash feedback → closeCam → handleBarcode
    ├── handleBarcode()         match / search / toast
    └── Settings modal          column remapping, data reload, reset
```

**External dependencies (CDN, no npm):**

| Library | Version | Purpose |
|---|---|---|
| `@zxing/library` | 0.19.1 | Barcode decoding fallback |
| `PapaParse` | 5.4.1 | CSV parsing |
| Google Fonts — Heebo | latest | Hebrew typeface |

---

## Development Notes

- All state lives in three `localStorage` keys: `yamit_data` (raw CSV text), `yamit_url`, `yamit_cols` (column mapping JSON).
- To reset completely: **Settings → מחק נתונים שמורים ואפס**, or clear `localStorage` from DevTools.
- The file uses `var` and `function` declarations throughout (no ES6+) to stay compatible with older Android WebView versions in use at some sites.
- `BarcodeDetector` formats list in `startNativeScan()` can be trimmed if only EAN/Code128 are needed — fewer formats = faster detection.
- ZXing's `listVideoInputDevices()` requires the page to already have camera permission; on first open the browser prompts before the list is available, which is why the native path handles its own `getUserMedia`.

---

## Roadmap / Known Gaps

- [ ] Manual torch (flashlight) toggle for dark warehouses
- [ ] Quantity adjustment — scan to increment/decrement stock and write back to Sheets via Apps Script
- [ ] Multi-sheet support (select tab from the published CSV dropdown)
- [ ] PWA manifest + service worker for full offline install
- [ ] Scan history (last N scanned items)
- [ ] BarcodeDetector polyfill for Safari (currently ZXing fallback handles it)

---

## Contributing

The project is intentionally a single file to stay deployable anywhere without tooling.  
Keep that constraint in mind: no bundler, no npm install, no framework.

1. Fork the repo
2. Create a feature branch (`git checkout -b feature/my-thing`)
3. Edit `index.html`
4. Test on a real mobile device over HTTPS (camera APIs don't work in simulators)
5. Open a PR with a description of what changed and why
