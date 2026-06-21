# Atera Water — Site Visit Capture (PWA)

A single, self-contained, **offline-first** Progressive Web App for Atera Water field
engineers. On a factory site (often with no signal), an engineer walks around, snaps a
photo, tags **what it is** + adds a note, then at the end exports **one `.zip`** and sends
it via WhatsApp / AirDrop / Mail. No backend, no login, no build step — plain HTML/CSS/JS,
everything runs on-device.

The exported zip matches the existing `site-visits/` folder format, so the downstream
gallery (`site-visits/index.html`) and the `deals-app` CRM importers consume it unchanged.

## Files

| File | Purpose |
|------|---------|
| `index.html` | The entire app — inline CSS + JS, no external dependencies. |
| `manifest.webmanifest` | PWA manifest (standalone display, name, icons). |
| `sw.js` | Service worker — pre-caches the app shell for offline launch. |
| `icon-192.png`, `icon-512.png` | App icons (also used as the iOS Home-Screen icon). |

## Run locally

Service workers don't run from `file://`, so serve over **http**:

```sh
cd site-visit-capture
python3 -m http.server 8080      # or: npx serve .
```

Open <http://localhost:8080>. On desktop, narrow the window to ~390px to see the phone
layout. To test on a real iPhone, you need **https** (see "Deploy" below) — the camera and
"Add to Home Screen" require a secure context.

## How it works

- **Cover screen** — Date (defaults to today), Site/Company, Engineer, Location. Company +
  date generate the export folder slug `YYYY-MM-DD-<company-slug>`.
- **Capture loop** — a running, timestamped, chronological log. **+ Photo** opens the native
  camera via `<input type="file" accept="image/*" capture="environment">` (deliberately *not*
  `getUserMedia`, which is buggy in iOS home-screen PWAs). After capture, a tag sheet lets
  you pick **"What is this?"** from the guided categories and/or type a note, auto-stamps the
  time, and offers optional one-tap GPS. **+ Video** and **+ Note** (note-only) are also
  supported. Every entry — photo/video blob + metadata — is written to **IndexedDB
  immediately**, so it survives reload / app kill.
- **Summary screen** — repeatable **Key Findings** and **Outstanding** bullets, plus a
  reminder to export before leaving site.
- **Finish & Export** — builds the zip in-browser with a hand-rolled **store-only ZIP
  writer** (no compression — JPEG/MP4 are already compressed; ~no dependencies), then calls
  the **Web Share API** (`navigator.share({ files: [zip] })`) to open the native share sheet
  for WhatsApp/AirDrop/Mail. On browsers without file-share support it falls back to a normal
  download.

### Output format

The zip's single top-level entry is the folder `YYYY-MM-DD-<slug>/` containing:

- `notes.md` — `### HH:MM — <Category>` blocks, each with `- <note>` lines and a
  `` - Photo saved: `<file>` `` (or `` Video saved: `<file>` ``) reference, followed by a
  `## Visit Summary` section (Key Findings / Outstanding).
- the media files, named `<HHMM>-<category-slug>-<shortid>.<ext>` and referenced by that
  exact name in `notes.md`.

Unzip straight into `site-visits/` and the gallery resolves every `./<folder>/<file>` path.

### Storage / offline notes

- Data lives in **IndexedDB** (blobs included) and persists across reloads and app kills.
- The service worker pre-caches the app shell, so the app **opens with zero network**.
- The app makes **no network calls at runtime** — no CDNs, fonts, or analytics.
- iOS may evict on-device storage under low-disk pressure (installed PWAs are exempt from
  the 7-day-inactivity rule but **not** from low-storage eviction). The app calls
  `navigator.storage.persist()` and nudges export-per-visit. Treat the device as a buffer,
  not long-term storage — **export each visit before leaving site.**

## Deploy (remaining step — handled by Adrian)

Hosting is **not** done here. The remaining step is to deploy this folder to **Vercel** as a
static PWA. HTTPS is required for iOS "Add to Home Screen" + camera. All manifest and
service-worker paths are **relative** (`./sw.js`, `./icon-512.png`), so the app works both
locally and once hosted at any base path.
