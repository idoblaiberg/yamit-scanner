# Label Snapshot — Design Spec
_Date: 2026-05-23_

## Context

The current scanner uses ZXing's continuous video stream (`decodeFromVideoDevice`) to auto-detect barcodes in real time. In practice this is unstable — motion blur, lighting, and angle cause missed reads. The new approach replaces continuous scanning with a deliberate **still-image capture**: the user aims the camera, taps to freeze a frame, and the app then runs both barcode decode and OCR on the static image. This is both more reliable for barcode reading and opens up label-text extraction as a fallback identification path.

A secondary outcome: when a barcode is captured but doesn't match the local inventory, the data is saved to a **pending barcodes queue** that can be exported as CSV and bulk-imported into Financit to enrich the product database.

---

## Decisions

| Decision | Choice |
|---|---|
| Relationship to existing scan | Replaces entirely — one mode only |
| Capture trigger | Tap anywhere on the viewfinder (ghost shutter button in corner as hint) |
| Barcode decode | ZXing on still image (same library, different API) |
| OCR engine | Tesseract.js — runs fully in-browser, offline, no API key |
| When results appear | After a 2-step flow: OCR preview first, then product result |
| Result layout | Image thumbnail + product data + OCR text (expanded modal) |
| No-match handling | Save to pending CSV queue; user can still manual-search |
| Financit integration | Export pending queue as CSV from settings panel |

---

## Three-Screen Flow

### Screen 1 — Viewfinder
- The existing camera overlay is repurposed: `<video>` shows the live feed as before.
- Remove the animated scan-line. Replace with a static corner-bracket frame (aim guide).
- Ghost shutter icon in the bottom-right corner as a visual hint.
- Entire screen is tappable → triggers capture.
- Close (✕) button top-left remains.

**On tap:**
1. Call `captureFrame()` — draw current video frame to a hidden `<canvas>`, export as base64 PNG.
2. Stop the video stream immediately (free camera resource).
3. Show a brief "מעבד…" spinner over the frozen frame.
4. Run in parallel:
   - `decodeBarcode(imageData)` — ZXing `decodeFromCanvas()`
   - `extractOCR(imageData)` — Tesseract.js `recognize()`
5. When both complete → navigate to Screen 2.

### Screen 2 — Extracted Data Preview (מה נמצא)
A new intermediate modal/panel (not a full page) showing:
- The captured image (thumbnail, full-width inside the panel)
- **Barcode section** (green): barcode value if found, or "לא נמצא ברקוד" if not
- **OCR section** (blue-tinted): raw extracted text from Tesseract, line-wrapped

One action button: **"🔍 חפש מוצר"**

On tap → calls `handleBarcode(barcodeValue)` if barcode was found, otherwise calls `doSearch(ocrText)` with the OCR text. Either way, the captured image and OCR text are kept in a session variable (`pendingScan`) for use in Screen 3.

### Screen 3 — Result
The existing result modal is extended:
- **Top row**: image thumbnail (44×44px) | product name + match indicator | 🔄 rescan button
- **Badges row**: stock badge, price (if scraping data available) — unchanged from today
- **New OCR section** at the bottom of the modal: small gray label "טקסט מהתווית" + the raw OCR string

**Rescan button (🔄)**: clears `pendingScan`, closes modal, reopens the camera overlay (Screen 1).

**No-match path**: if `handleBarcode`/`doSearch` finds no results:
- Show the normal "לא נמצא" toast
- BUT keep Screen 2 open (image + extracted data visible)
- Add a secondary action: **"💾 שמור ברקוד בהמתנה"**
  - Appends `{ barcode, ocrText, imageDataUrl, timestamp }` to `pendingBarcodes[]` in localStorage
  - Updates a badge counter on a new "ממתינים" (pending) button in the header
  - The manual search field remains active so the user can still find the product by name

---

## Pending Barcodes Queue

### Data shape (per entry)

```json
{
  "barcode": "7290000123456",
  "ocrText": "שמן זית כתית 750 מ\"ל",
  "timestamp": 1748000000000,
  "resolvedSku": null
}
```

Stored in `localStorage['yamit_pending']` as a JSON array. **Images are not stored** — a full-resolution PNG is 200–800KB; even 10 entries would risk hitting localStorage's 5MB limit. The captured image is kept only in the `pendingScan` session variable (memory) for display during the current scan session and discarded when the user moves on.

When `pendingScan` is active (a snapshot was taken and not yet cleared), any product modal that opens — whether from auto-match or manual search — shows a **"🔗 קשר ברקוד למוצר זה"** button. Tapping it sets `resolvedSku` on the most recent unresolved pending entry and shows a toast confirming the link. `pendingScan` is then cleared.

### Export
A **"ייצא ממתינים"** button in the Settings panel exports the queue as a CSV:
```
barcode,ocrText,resolvedSku,timestamp
7290000123456,"שמן זית כתית 750 מ\"ל",1234,1748000000000
```
After export, the queue is cleared (or the user is prompted).

---

## Technical Notes

### ZXing on still image
```js
const reader = new ZXing.BrowserMultiFormatReader();
const result = await reader.decodeFromCanvas(canvasElement);
```
Uses the same `@0.19.1` CDN already loaded. No new dependency.

### Tesseract.js
Add CDN script tag:
```html
<script src="https://cdn.jsdelivr.net/npm/tesseract.js@5/dist/tesseract.min.js"></script>
```
Run with Hebrew + English language data:
```js
const { data: { text } } = await Tesseract.recognize(imageDataUrl, 'heb+eng');
```
First run downloads language data (~10MB) and caches it in the browser. Subsequent runs are fast.

### captureFrame()
```js
function captureFrame() {
  const canvas = document.getElementById('snapCanvas'); // hidden canvas
  canvas.width = video.videoWidth;
  canvas.height = video.videoHeight;
  canvas.getContext('2d').drawImage(video, 0, 0);
  return canvas.toDataURL('image/png');
}
```

### Session state
```js
let pendingScan = null;
// shape: { imageDataUrl, barcode, ocrText }
```
Cleared on new scan or after result modal closes normally.

---

## Files to Modify

| File | Change |
|---|---|
| `index.html` | All changes — single-file app. Add Tesseract.js CDN, hidden `<canvas id="snapCanvas">`, new Screen 2 panel, extended result modal, pending queue logic, settings export button |
| `.gitignore` | Already updated (`.superpowers/` added) |

**Existing functions to reuse:**
- `openCam()` (`index.html` ~line 799) — keep camera init, replace decode loop with tap handler
- `handleBarcode(val)` (~line 828) — call unchanged; pass barcode value from still
- `doSearch(query)` (~line 578) — call with OCR text as fallback
- `openResult(match)` (~line 739) — extend to accept optional `pendingScan` data
- `showToast(msg)` — reuse for "saved to pending" confirmation
- `makeCard()` (~line 608) — unchanged

---

## Verification

1. Open app in mobile browser (or Chrome DevTools mobile emulation).
2. Tap the camera button → viewfinder opens, no scan line.
3. Tap anywhere on the video → frame freezes, spinner shows briefly.
4. Screen 2 appears: image thumbnail, barcode value (or "לא נמצא"), OCR text.
5. Tap "חפש מוצר" → product modal opens with image thumbnail + OCR section at bottom.
6. Tap 🔄 rescan → camera reopens cleanly.
7. With a barcode that's not in inventory: Screen 2 stays open, "שמור ברקוד בהמתנה" appears.
8. Save a pending barcode → header badge increments.
9. Go to Settings → "ייצא ממתינים" → CSV downloads with correct columns.
10. Tesseract language data loads on first use (check Network tab) and is cached on second use.
