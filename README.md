# Yamit Scanner — סורק מלאי יאמיט

A mobile-first barcode scanner and inventory lookup tool for Yamit surf store staff. No backend, no build step — a single HTML file served via GitHub Pages.

**Live app:** https://idoblaiberg.github.io/yamit-scanner/

---

## Features

- **Barcode scanning** — point camera at any product barcode; instant lookup
- **Text search** — partial match on name, SKU, or alternative number
- **Stock status** — color-coded badges (green / gray / red) from the warehouse CSV
- **Price enrichment** — regular price, sale price, features, and product URL from a daily scraping report
- **Offline-capable** — both data sources cached in `localStorage` after first load
- **Hebrew / RTL** — fully localized for store staff

---

## Quick Start (Store Staff)

### First load

1. Open https://idoblaiberg.github.io/yamit-scanner/ on your phone
2. Paste the Google Sheets inventory URL → tap **טען מהקישור**
3. The app stores data locally — you won't need to reload it daily

### Loading price data

1. Tap ⚙️ in the top-right corner
2. Scroll to **נתוני מחירים (סקרייפינג)**
3. Paste the scraping report Google Sheets URL → tap **שמור וטען נתוני מחירים**
4. A green count confirms how many items were enriched

### Scanning

- Tap the 📷 button → point the camera at a barcode → result appears automatically
- Or type in the search bar (name, SKU, or manufacturer number)

### Refreshing data

- Go to ⚙️ Settings → tap **שמור וטען מחדש** to pull the latest inventory
- Tap **שמור וטען נתוני מחירים** to pull the latest scraping report

---

## Data Sources

| Source | Format | Refresh |
|---|---|---|
| Inventory CSV | Google Sheets → Publish as CSV | On demand |
| Scraping report | Google Sheets → Publish as CSV | Every morning (automated) |

See [docs/DATA_SOURCES.md](docs/DATA_SOURCES.md) for full column specs and join logic.

---

## Development

### Prerequisites

- Any text editor
- A local HTTP server (camera requires HTTPS or localhost)

### Local dev

```bash
# Python 3
python3 -m http.server 8080

# Then open http://localhost:8080
```

The entire app lives in `index.html` — no build step, no dependencies to install.

See [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) for design decisions and [docs/DEPLOYMENT.md](docs/DEPLOYMENT.md) for publishing to GitHub Pages.

---

## Repository Structure

```
yamit-scanner/
├── index.html              # The app — HTML + CSS + JS, self-contained
├── README.md
├── CHANGELOG.md
├── .gitignore
├── docs/
│   ├── ARCHITECTURE.md     # Technical decisions and code structure
│   ├── DATA_SOURCES.md     # CSV format specs for both data sources
│   ├── DEPLOYMENT.md       # GitHub Pages setup and update workflow
│   └── BRIEF.md            # Original product brief
└── csv/
    └── sample/
        └── inventory-sample.csv   # Anonymized sample data for testing
```

---

## Tech Stack

| Concern | Solution |
|---|---|
| Barcode scanning | [ZXing-js](https://github.com/zxing-js/library) `@0.19.1` via CDN |
| CSV parsing | [PapaParse](https://www.papaparse.com/) `5.4.1` via CDN |
| Persistence | `localStorage` |
| Font | Heebo (Google Fonts) |
| Hosting | GitHub Pages |
| Backend | None |

---

## License

Internal tool — Yamit surf store. Not for public distribution.
