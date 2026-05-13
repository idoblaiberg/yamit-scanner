# Data Sources

The app consumes two independent CSV files, both published from Google Sheets. They are joined at render time on a shared SKU field.

---

## Source 1 — Inventory CSV (warehouse stock)

**Updated:** on demand (exported from the warehouse management system)
**Loaded via:** Settings → מקור נתונים, or file upload on first launch
**localStorage key:** `yamit_data`

### Column headers

```
מ.במחסן,מחס,מספר פריט,שם פריט,מספר חליפי,***,כמות במלאי
```

| Column | Default mapping key | Description |
|---|---|---|
| `מספר פריט` | `cols.id` | **Primary identifier** — matched against barcodes and used as join key |
| `שם פריט` | `cols.name` | Product name — displayed on cards and in the modal |
| `מספר חליפי` | `cols.alt` | Manufacturer / alternative barcode — secondary match |
| `כמות במלאי` | `cols.stock` | Warehouse stock quantity; can be negative (deficit) |
| `מ.במחסן`, `מחס`, `***` | — | Ignored |

Column names are configurable in Settings → **מיפוי עמודות**.

### Filtering rules

Rows are excluded when:
1. Both `שם פריט` and `מספר פריט` are empty
2. `שם פריט` matches the pattern `/^\*+.*\*+$/` (category header rows like `******* סירות לייזר **********`)

### Sample rows

```csv
מ.במחסן,מחס,מספר פריט,שם פריט,מספר חליפי,***,כמות במלאי
,0,5,שובר מתנה (כמות במינוס) 50שנה,,,2.00
,0,50,"משלוח י.ר עד 20 ק""ג ..........",,,-5.00
,0,1,******* סירות לייזר **********,,,
,0,1000000000,סירת לייזר E6 RADIAL ILCA,E6IBFR4,,
9841013.00,0,1006,סירת לייזר PICO  זוגית,14000,,
,0,111608,רצועת איזון מרופדת לייזר ILCA,E6ID02,,-2.00
```

---

## Source 2 — Scraping Report (prices & features)

**Updated:** every morning automatically
**Loaded via:** Settings → נתוני מחירים (סקרייפינג)
**localStorage key:** `yamit_scrape_data`

### Column headers

```
ענף ספורט,קטגוריה,שם מוצר,תכונה 1,תכונה 2,מק"ט,מחיר רגיל (₪),מחיר מבצע (₪),מלאי,URL מוצר
```

| Column | Default mapping key | Description |
|---|---|---|
| `מק"ט` | `scrapeCols.sku` | **Join key** — matched against `cols.id` from the inventory CSV |
| `מחיר רגיל (₪)` | `scrapeCols.price` | Regular / list price |
| `מחיר מבצע (₪)` | `scrapeCols.salePrice` | Sale price; empty or equal to regular price when no sale is active |
| `מלאי` | `scrapeCols.stock` | Online stock / availability status |
| `תכונה 1` | `scrapeCols.feature1` | Product spec or feature (first) |
| `תכונה 2` | `scrapeCols.feature2` | Product spec or feature (second) |
| `URL מוצר` | `scrapeCols.url` | Direct link to product page |
| `ענף ספורט`, `קטגוריה`, `שם מוצר` | — | Metadata — not currently displayed |

Column names are configurable in Settings → **מיפוי עמודות – דוח מחירים**.

---

## Join Logic

The scraping report is parsed into an in-memory lookup map:

```js
scrapeMap[sku] = row   // keyed by scrapeCols.sku, trimmed
```

At render time:
```js
var enrichment = scrapeMap[String(product[cols.id] || '').trim()];
```

- Match is **exact string** after `.trim()` on both sides
- If no match: product card shows no price; result modal shows no "נתוני אתר" section
- If matched: card shows price inline; modal shows full enrichment section

### Price display rules

| Condition | Card display | Modal display |
|---|---|---|
| Sale price exists and differs from regular | Sale price (green) + regular (struck-through) | Both rows |
| Regular price only | Regular price | Regular price row |
| No scraping match | Nothing | No "נתוני אתר" section |

---

## Google Sheets → CSV URL

Both sources are loaded from a **"Publish to web → CSV"** URL, which looks like:

```
https://docs.google.com/spreadsheets/d/SHEET_ID/pub?output=csv&gid=SHEET_GID
```

This URL is public, CORS-friendly, and returns plain CSV text — no authentication required.

To get this URL from Google Sheets: **File → Share → Publish to web → select sheet → CSV → Publish**.
