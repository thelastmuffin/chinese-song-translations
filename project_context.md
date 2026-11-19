# Chinese Song Translations — Project Context

Drop this file into the root of your `chinese-song-translations` repository so Antigravity (or any coding agent) immediately has full context on the app's architecture, user preferences, and Google Sheets sync setup.

## 1. Project Overview & Links
- **GitHub Repo:** `https://github.com/thelastmuffin/chinese-song-translations`
- **Live GitHub Pages URL:** `https://thelastmuffin.github.io/chinese-song-translations/`
- **Main File:** `index.html` — a single self-contained HTML file (Tailwind CSS `3.4.17` via CDN, `Inter` + `Noto Sans SC` fonts, vanilla JS, no build step) so GitHub Pages serves it directly.
- **Google Sheet Database:** [Chinese Song Translations](https://docs.google.com/spreadsheets/d/1iYA50aGCyZnFS32ntwzwj108IQRWNs8TU3Yn2VYTHxo/edit) (Spreadsheet ID: `1iYA50aGCyZnFS32ntwzwj108IQRWNs8TU3Yn2VYTHxo`)
- **Gemini Song-Request Chat:** `https://gemini.google.com/app/472a44fdf6af0b15`

## 2. Division of Responsibilities
- **Gemini Chat:** Translates requested Chinese songs (Simplified Chinese, Pinyin with tone marks, English) and appends them directly to the `Songs` and `Lyrics` tabs of the Google Sheet.
- **Antigravity:** Handles UI/UX changes, new features, refactoring, and pushing commits to `main` on GitHub (which automatically redeploys GitHub Pages).

## 3. User Preferences & UI Rules
- **Theme:** Default to **dark mode** (`<html lang="en" class="dark">` with Tailwind `darkMode: 'class'`), keeping a light/dark toggle in the header.
- **Line Order:** Always display **Simplified Chinese → Pinyin → English** (Pinyin strictly in between Chinese and English in both Study View and Table View).
- **Mobile & Desktop Shortcuts:** Maintain PWA/home-screen meta tags (`apple-mobile-web-app-capable`, `apple-mobile-web-app-status-bar-style`, `theme-color`) and keep the **✨ Ask Gemini** button accessible on both mobile and desktop headers.
- **Browser Safety:** Avoid `alert()`, `confirm()`, and `prompt()` (use in-page modals and toasts instead). Keep `localStorage` calls wrapped in `try/catch` and preserve clipboard `execCommand('copy')` fallback.

## 4. Google Sheets Cloud Sync Schema
The app syncs from a deployed Google Apps Script Web App URL (`https://script.google.com/macros/s/.../exec`) attached to the Google Sheet.

### Tab 1: `Songs`
Exact Row 1 headers (with optional `coverUrl` and `ytMusicUrl` columns):
`id | titleZh | titleEn | singer | singerZh | album | year | lyricist | composer | tags | sourceLabel | sourceUrl | favorite | coverUrl | ytMusicUrl`

### Tab 2: `Lyrics`
Exact Row 1 headers:
`songId | section | lineOrder | zh | py | en`

### Companion Google Apps Script (`doGet`)
Supports both standard `fetch()` JSON and `?callback=` JSONP fallback:

```javascript
function doGet(e) {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const songRows = ss.getSheetByName("Songs").getDataRange().getValues();
  const lyricRows = ss.getSheetByName("Lyrics").getDataRange().getValues();
  const sHead = songRows[0];
  const lyricsBySong = {};
  for (let i = 1; i < lyricRows.length; i++) {
    const row = lyricRows[i];
    if (!row[0]) continue;
    const sid = String(row[0]).trim();
    const sec = String(row[1] || "Lyrics").trim();
    if (!lyricsBySong[sid]) lyricsBySong[sid] = [];
    let secObj = lyricsBySong[sid].find(s => s.name === sec);
    if (!secObj) { secObj = { name: sec, lines: [] }; lyricsBySong[sid].push(secObj); }
    secObj.lines.push({ zh: String(row[3] || ""), py: String(row[4] || ""), en: String(row[5] || "") });
  }
  const songs = [];
  for (let i = 1; i < songRows.length; i++) {
    const r = songRows[i];
    if (!r[0]) continue;
    const obj = {};
    sHead.forEach((h, idx) => obj[h] = r[idx]);
    const sid = String(obj.id).trim();
    songs.push({
      id: sid,
      titleZh: String(obj.titleZh || ""),
      titleEn: String(obj.titleEn || ""),
      singer: String(obj.singer || ""),
      singerZh: String(obj.singerZh || ""),
      album: String(obj.album || ""),
      year: obj.year ? Number(obj.year) : null,
      lyricist: String(obj.lyricist || ""),
      composer: String(obj.composer || ""),
      tags: String(obj.tags || "").split(",").map(t => t.trim()).filter(Boolean),
      sourceLabel: String(obj.sourceLabel || "Lyrics Source"),
      sourceUrl: String(obj.sourceUrl || ""),
      coverUrl: String(obj.coverUrl || ""),
      ytMusicUrl: String(obj.ytMusicUrl || ""),
      favorite: String(obj.favorite).toUpperCase() === "TRUE",
      createdAt: Date.now() - i * 1000,
      bookmarkedLines: [],
      sections: lyricsBySong[sid] || []
    });
  }
  const json = JSON.stringify(songs);
  const cb = e && e.parameter && e.parameter.callback;
  if (cb) return ContentService.createTextOutput(cb + "(" + json + ")").setMimeType(ContentService.MimeType.JAVASCRIPT);
  return ContentService.createTextOutput(json).setMimeType(ContentService.MimeType.JSON);
}
```

## 5. Client-Side Data Model & `localStorage` Keys
- `chinese_song_translations_library_v1` — Array of song objects (including user `favorite` flags, optional `coverUrl`, and `bookmarkedLines` like `"0-2"`).
- `chinese_song_translations_theme_v1` — `'dark'` or `'light'`.
- `chinese_song_translations_sync_url_v1` — User's deployed Google Apps Script `/exec` URL.
- `chinese_song_translations_cover_cache_v1` — Cached auto-fetched album cover URLs keyed by song `id`.
- `chinese_song_translations_sidebar_v1` — `'collapsed'` or `'open'` for desktop sidebar minimize state.

### Song Object Shape
```json
{
  "id": "sun-yanzi-wo-huai-nian-de",
  "titleZh": "我怀念的",
  "titleEn": "What I Miss",
  "singer": "Sun Yanzi",
  "singerZh": "孙燕姿",
  "album": "Against the Light (逆光)",
  "year": 2007,
  "lyricist": "Yao Ruolong (姚若龙)",
  "composer": "Lee Shih Shiong (李偲菘)",
  "sourceLabel": "KKBOX Official Lyrics",
  "sourceUrl": "https://www.kkbox.com/sg/en/song/-qn2N4CALDIxDm5Os5",
  "tags": ["Ballad", "Nostalgic", "Mandopop"],
  "favorite": true,
  "createdAt": 1763545600000,
  "bookmarkedLines": [],
  "sections": [
    {
      "name": "Verse 1",
      "lines": [
        { "zh": "...", "py": "...", "en": "..." }
      ]
    }
  ]
}
```

## 6. Current Features to Preserve
- Main Song Navigation screen (compact album-cover card grid: 3 per row on mobile, 5 per row on desktop, plus a **By Album** cover-only view that opens a tracklist on click, a **✨ Discover** recommendations tab with personalized & Mandopop essentials picks, Album dropdown filter, live search, singer/tag/favorites filters, and sort) as the home view, plus hash history (`#song/<id>`) for browser/Android Back button support
- Collapsible sidebar on both mobile and desktop (`#sidebar`, `#sidebarToggleBtn`, `#collapseSidebarDesktopBtn`, `#mobileBackdrop`) with compact song thumbnails, Library Home & Discover shortcuts, active album filter banner, and bottom-pinned **Sheet Sync** + **Dark/Light Mode** controls
- Red & yellow **乐♪** logo (`logo.svg`, `icon-192.png`, `icon-512.png`, `manifest.json`) wired into the header, favicon, and PWA shortcuts
- Top header featuring clickable Home brand and primary **✨ Ask Gemini** modal with **`Single Song` | `Full Album` | `Get Recs`** modes (configurable 3 / 5 / 10 tracks per batch for full albums, defaulting to 10; plus 1-tap **`✨ Retranslate`** on every song page that instructs Gemini to overwrite that `songId`'s rows cleanly; auto-copy & open Gemini chat for Android & Windows; Import/Export removed)
- Clickable artist names across Song Reader, Album view, Home cards, and Discover opening an **Artist Explorer** modal (in-library songs, live iTunes Search popular tracks with 1-tap YouTube Music / Translate buttons, and direct YouTube Music artist link)
- "Similar Songs to Explore" recommendation strip at the bottom of every Song Reader page
- Album covers / song thumbnails visible on the navigation grid, song reader header (with ambient blur backdrop), and sidebar — auto-fetched via iTunes Search JSONP with `coverUrl` override and stylized gradient fallbacks
- YouTube Music quick-play links on every song card, album tracklist row, and song reader page — uses explicit `ytMusicUrl` if provided or auto-generates a `https://music.youtube.com/search?q=...` deep link (opens YouTube Music app on Android & web on Windows)
- Study View (stacked cards: Chinese → Pinyin → English) and Table View (3 columns)
- Language layer toggles (`中文`, `Pinyin`, `English`) with at-least-one-visible guard
- Font size controls (`A-`, `A`, `A+`) and section jump pills
- Click-any-line study bookmarking
- Edit / Delete song modals (plus manual add option inside Ask Gemini modal) and Copy Markdown
- Sheet Sync modal (auto-sync on load, `fetch` + JSONP fallback, Test URL button, Copy Apps Script button)

## 7. Pre-Loaded Songs (Verified Sources)
1. **Sun Yanzi (孙燕姿) — 《原来你什么都不要》 (Nothing You Want)**, *Start (自选集)*, 2002 — [KKBOX](https://www.kkbox.com/sg/en/song/8rITqbZ059bDR-Rdwg)
2. **Sun Yanzi (孙燕姿) — 《我怀念的》 (What I Miss)**, *Against the Light (逆光)*, 2007 — [KKBOX](https://www.kkbox.com/sg/en/song/-qn2N4CALDIxDm5Os5)
