# Deployment

The app is a single `index.html` file served from **GitHub Pages**.

Live URL: **https://idoblaiberg.github.io/yamit-scanner/**

---

## Why GitHub Pages

- Free HTTPS hosting — required for `getUserMedia` (camera) on iOS/Android
- Zero configuration — push `index.html` to `main`, it's live within ~60 seconds
- No server, no build pipeline

---

## Initial Setup (one-time)

1. Push the repository to GitHub
2. Go to **Settings → Pages**
3. Under **Source**, select **Deploy from a branch** → `main` → `/ (root)`
4. Save — GitHub will provide the live URL

---

## Deploying an Update

```bash
git add index.html
git commit -m "describe the change"
git push origin main
```

GitHub Pages rebuilds automatically. Allow ~60 seconds for the CDN to propagate.

---

## Testing Locally

Camera access (`getUserMedia`) is blocked on `file://` URLs. Use a local HTTP server:

```bash
# Python 3
python3 -m http.server 8080

# Node (if npx is available)
npx serve .
```

Then open **http://localhost:8080** in your browser.

For iOS testing, use a tool like [ngrok](https://ngrok.com/) or push to a branch and test via the GitHub Pages URL directly — iOS Safari requires HTTPS for camera access.

---

## Branch Workflow

Feature work happens on `claude/` branches (created by Claude Code). To ship:

```bash
# Review changes
git diff main...claude/<branch-name>

# Merge to main
git checkout main
git merge claude/<branch-name>
git push origin main
```

Or open a pull request on GitHub for review before merging.

---

## CDN Dependencies

The app depends on two CDN-hosted libraries. If CDN availability is ever a concern, download and commit them:

| Library | CDN URL |
|---|---|
| ZXing `@0.19.1` | `https://unpkg.com/@zxing/library@0.19.1/umd/index.min.js` |
| PapaParse `5.4.1` | `https://cdnjs.cloudflare.com/ajax/libs/PapaParse/5.4.1/papaparse.min.js` |

To self-host, download both files, place them in an `assets/js/` folder, and update the `<script src="...">` tags in `index.html`.
