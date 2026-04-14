Dum# Converting Ear Tuner to an Electron Desktop App

## What Is Electron?

Electron bundles **Chromium** (the rendering engine) + **Node.js** (system access) into a single executable. Your UI is still HTML/CSS/JS — it just runs in an embedded browser window rather than a phone browser. The two halves of an Electron app are:

| Process | What it is | Analogous to |
|---|---|---|
| **Main process** | Node.js script; controls windows, menus, system tray, native dialogs | "The OS layer" |
| **Renderer process** | Chromium tab running your HTML/CSS/JS | The phone browser |
| **Preload script** | A bridge that exposes a limited, safe API from main → renderer | iOS WKWebView message handler |

The renderer cannot call Node.js APIs directly (for security). The preload script uses Electron's `contextBridge` to expose only what the renderer needs — e.g., `window.electronAPI.saveSettings(data)`.

---

## What Would Actually Change

### What stays the same
- All HTML structure (none of it is iOS-specific)
- All CSS (safe-area vars become `0` on desktop — no harm done)
- All game logic, state machine, round logic, scoring
- Web Audio API (Chromium supports it fully, no iOS resume hacks needed)
- The soundfont loading approach (still works; can be local files instead of CDN)

### What gets removed
- `<meta name="apple-mobile-web-app-capable">` and iOS-specific meta tags
- `env(safe-area-inset-*)` fallbacks can be simplified to `0`
- `pageshow` BFCache handler (no BFCache in Electron)
- `webkitAudioContext` fallback (Chromium always has `AudioContext`)
- Touch gesture handling — replaced with keyboard equivalents
- `ensureAudio()` async resume dance (no iOS gesture requirement)

### What gets restructured
- **localStorage → `electron-store`** — a small npm package that persists JSON to a proper user-data file (`AppData/Roaming/ear-tuner/`) instead of browser storage. The API is nearly identical. Settings and stats survive app reinstalls.
- **Soundfonts → local files** — copy them out of the CDN URL into the app bundle; no network dependency
- **Google Fonts → local** — bundle `Inconsolata` and `Nunito` as font files; no network dependency
- **Single file → multiple files** — Electron needs at minimum `main.js` + a renderer entry point; the benefit is that each concern lives in its own file

### What gets added
- Native app menu (File, Edit, Help)
- Keyboard shortcuts (Space = replay, ↑ = "second note higher", ↓ = "second note lower", → = continue)
- Window state persistence (size/position remembered across launches)
- `package.json` + build tooling (`electron-builder`) to produce a `.exe` installer

---

## Recommended File and Folder Structure

```
ear-tuner/
├── package.json               ← npm config, electron/electron-builder deps, build config
├── package-lock.json
│
├── src/                       ← all source code
│   ├── main/                  ← Main process (Node.js)
│   │   ├── main.js            ← Entry: creates BrowserWindow, registers app menu
│   │   ├── menu.js            ← Native menu definition
│   │   └── windowState.js     ← Persists window size/position via electron-store
│   │
│   ├── preload/
│   │   └── preload.js         ← contextBridge — the only bridge between main & renderer
│   │
│   └── renderer/              ← Renderer process (Chromium — your app)
│       ├── index.html         ← Main window HTML (≈ current index.html, restructured)
│       ├── styles/
│       │   └── app.css        ← Extracted from the current <style> block
│       ├── js/
│       │   ├── app.js         ← Top-level init, state machine, event wiring
│       │   ├── audio.js       ← AudioContext, ensureAudio(), playBothNotes(), chimes
│       │   ├── settings.js    ← Settings read/write, settings overlay UI
│       │   ├── stats.js       ← Per-note stats, scoring, regression logic
│       │   ├── rounds.js      ← Round logic, note selection, difficulty stepping
│       │   └── ui.js          ← DOM helpers: renderProgress(), setBg(), overlays
│       └── assets/
│           ├── fonts/
│           │   ├── Inconsolata-*.woff2
│           │   └── Nunito-Bold.woff2
│           ├── sounds/        ← Soundfont JS files (currently from CDN)
│           │   ├── violin-mp3.js
│           │   ├── viola-mp3.js
│           │   └── ...
│           └── icons/
│               ├── Ear_Pinhead_icon.svg
│               ├── icon.png   ← Windows .exe icon (256×256 png)
│               └── icon.ico   ← Windows taskbar icon
│
├── dist/                      ← electron-builder output (.exe installer) — gitignored
├── .gitignore
└── research/                  ← you are here
    └── converting_to_electron.md
```

> [!tip] Why split the JS into modules?
> Each JS file has one responsibility. `audio.js` doesn't know about the DOM. `stats.js` doesn't know about audio. This maps directly to Swift classes/structs at porting time, and it's also just easier to navigate. The current single-file approach was fine for a phone PWA — for a desktop app with a build step, modules are the right call.

---

## The Preload Bridge (What It Exposes)

The renderer cannot call `require('electron-store')` directly. The preload script exposes a typed API:

```js
// src/preload/preload.js
const { contextBridge, ipcRenderer } = require('electron');

contextBridge.exposeInMainWorld('electronAPI', {
  getSettings:  ()       => ipcRenderer.invoke('store:getSettings'),
  saveSettings: (data)   => ipcRenderer.invoke('store:saveSettings', data),
  getStats:     ()       => ipcRenderer.invoke('store:getStats'),
  saveStats:    (data)   => ipcRenderer.invoke('store:saveStats', data),
  getLog:       ()       => ipcRenderer.invoke('store:getLog'),
  saveLog:      (data)   => ipcRenderer.invoke('store:saveLog', data),
});
```

In the renderer, `settings.js` calls `window.electronAPI.saveSettings(data)` instead of `localStorage.setItem(...)`. The shape of the data doesn't change — only the storage mechanism does.

> [!question] Should we keep localStorage as a fallback (so the PWA still works)?
> We could write a thin adapter: `StorageAdapter.saveSettings(data)` calls either `window.electronAPI` (if present) or `localStorage` (if not). This lets the same renderer code serve both the PWA and the Electron build with no branching in business logic.

> [!claude] Tentatively, yes. I have a follow question that I'm asking in the terminal.

---

## Keyboard Shortcuts (replacing swipe gestures)

| Gesture (phone) | Keyboard (desktop) |
|---|---|
| Swipe up — "second note is higher" | `↑` or `K` |
| Swipe down — "second note is lower" | `↓` or `J` |
| Tap lower half — replay | `Space` or `R` |
| Swipe right — Continue | `→` or `Enter` |
| Swipe left — Retry | `←` or `Backspace` |

Keyboard events are handled in `app.js` alongside (or instead of) the touch handlers.

> [!question] Should keyboard shortcuts be configurable, or hardcoded?
> Hardcoded is fine for v1. The notes app doesn't need remappable keys.

> [!claude] Hardcoded is fine. Your suggested mappings are good.

---

## Storage: localStorage vs. electron-store

|                    | `localStorage` (current)   | `electron-store` (Electron)             |
| ------------------ | -------------------------- | --------------------------------------- |
| Location           | Browser's internal storage | `AppData\Roaming\ear-tuner\config.json` |
| Survives reinstall | No                         | Yes (if same user profile)              |
| Human-readable     | No                         | Yes — plain JSON file                   |
| Size limit         | ~5MB                       | Unlimited                               |
| API                | `setItem` / `getItem`      | `store.set()` / `store.get()`           |

The current localStorage keys (`vio4-settings`, `vio4-stats`, `vio4-log`) map directly to top-level keys in the electron-store JSON file. Migration is straightforward.

---

## Build Tooling: electron-builder

`electron-builder` packages the app into a Windows installer (`.exe` / NSIS) with one command:

```json
// package.json (excerpt)
{
  "name": "ear-tuner",
  "version": "1.0.0",
  "main": "src/main/main.js",
  "scripts": {
    "start": "electron .",
    "build": "electron-builder --win"
  },
  "dependencies": {
    "electron-store": "^10.0.0"
  },
  "devDependencies": {
    "electron": "^31.0.0",
    "electron-builder": "^24.0.0"
  },
  "build": {
    "appId": "app.fiddle.ear-tuner",
    "productName": "Ear Tuner",
    "directories": { "output": "dist" },
    "win": {
      "target": "nsis",
      "icon": "src/renderer/assets/icons/icon.ico"
    }
  }
}
```

`npm run start` launches in dev. `npm run build` produces `dist/Ear Tuner Setup 1.0.0.exe`.

---

## What This Is NOT

This plan does **not** propose:

- A React/Vue/framework rewrite — still vanilla JS
- A full refactor of the game logic — the logic is solid, it just moves into separate files
- An auto-updater (can be added later via `electron-updater`)
- A Mac build (easy to add later; `"mac": { "target": "dmg" }`)

---

## Summary: What Changes, and Why

| Area              | Current (PWA)              | Electron                                                      |
| ----------------- | -------------------------- | ------------------------------------------------------------- |
| File organization | 1 HTML file                | ~10 files across `src/main/`, `src/preload/`, `src/renderer/` |
| Storage           | `localStorage`             | `electron-store` (JSON in AppData)                            |
| Audio context     | Async resume dance for iOS | Direct `new AudioContext()`, always works                     |
| Fonts             | Google Fonts CDN           | Bundled `.woff2` files                                        |
| Soundfonts        | CDN URL                    | Local files in `assets/sounds/`                               |
| Input             | Touch + swipe              | Mouse + keyboard                                              |
| iOS-specific code | Several workarounds        | Removed                                                       |
| Packaging         | GitHub Pages static host   | `electron-builder` → `.exe` installer                         |

The migration is mechanical, not a rewrite. The hardest part is extracting the inline JS into coherent modules — which is also the most valuable part, since it sets up a clean structure for the future Swift port.

---

## Dual-Target: Electron + PWA from One Codebase

Yes — this is a well-worn pattern and it works cleanly here. The renderer code (HTML/CSS/JS) is standard web code; it just needs to not call Electron-specific APIs directly. The structure is: **same renderer source, two packaging paths**.

### The Core Principle: One Source, Two Outputs

```
src/renderer/       ← single source of truth for all UI code
     ↓                        ↓
npm run build:electron    npm run build:pwa
     ↓                        ↓
dist/electron/            dist/web/
  Ear Tuner Setup.exe       index.html + assets (deploy to GitHub Pages)
```

No forking. No "fix it in two places." The renderer source is identical for both targets.

### The Only Split: StorageAdapter

The one place where Electron and browser diverge is storage. The fix is to isolate it in one file — `storage.js` — which both targets share but implement differently via environment detection:

```js
// src/renderer/js/storage.js
const isElectron = () => typeof window !== 'undefined' && !!window.electronAPI;

export const StorageAdapter = {
  async getSettings() {
    return isElectron()
      ? window.electronAPI.getSettings()
      : JSON.parse(localStorage.getItem('vio4-settings'));
  },
  async saveSettings(data) {
    isElectron()
      ? window.electronAPI.saveSettings(data)
      : localStorage.setItem('vio4-settings', JSON.stringify(data));
  },
  async getStats() {
    return isElectron()
      ? window.electronAPI.getStats()
      : JSON.parse(localStorage.getItem('vio4-stats'));
  },
  async saveStats(data) {
    isElectron()
      ? window.electronAPI.saveStats(data)
      : localStorage.setItem('vio4-stats', JSON.stringify(data));
  },
  // same pattern for vio4-log
};
```

Every other JS module calls `StorageAdapter.saveSettings(data)` — never `localStorage` directly, never `window.electronAPI` directly. That's the whole pattern.

### Updated File Structure (Dual-Target)

```
ear-tuner/
├── package.json               ← both electron and pwa build scripts
│
├── src/
│   ├── main/                  ← Electron only (not deployed to web)
│   │   ├── main.js
│   │   ├── menu.js
│   │   └── windowState.js
│   │
│   ├── preload/               ← Electron only
│   │   └── preload.js
│   │
│   └── renderer/              ← Shared — deployed unchanged to both targets
│       ├── index.html
│       ├── manifest.json      ← PWA only (harmless if present in Electron build)
│       ├── sw.js              ← Service worker for offline PWA support
│       ├── styles/
│       │   └── app.css
│       ├── js/
│       │   ├── app.js
│       │   ├── audio.js
│       │   ├── settings.js
│       │   ├── stats.js
│       │   ├── rounds.js
│       │   ├── ui.js
│       │   └── storage.js     ← THE adapter — the only env-aware file
│       └── assets/
│           ├── fonts/
│           ├── sounds/
│           └── icons/
│
├── dist/
│   ├── electron/              ← .exe output, gitignored
│   └── web/                   ← static site output, deploy to GitHub Pages
│
└── scripts/
    └── build-pwa.js           ← copies src/renderer/ → dist/web/, injects cache manifest
```

### Build Scripts

```json
// package.json scripts
{
  "start":           "electron .",
  "build:electron":  "electron-builder --win",
  "build:pwa":       "node scripts/build-pwa.js",
  "build:all":       "npm run build:electron && npm run build:pwa"
}
```

`build-pwa.js` is a ~30-line Node script that copies `src/renderer/` to `dist/web/` and generates a cache manifest for the service worker. No webpack, no bundler needed — the renderer uses native ES modules (`<script type="module">`), which both Chromium and modern mobile browsers support.

### What's PWA-Only vs. Electron-Only

| File / Feature | PWA | Electron | Notes |
|---|---|---|---|
| `manifest.json` | ✓ required | harmless | Electron ignores it |
| `sw.js` (service worker) | ✓ required for offline | not registered | `navigator.serviceWorker` is a no-op in Electron |
| iOS `<meta>` tags | ✓ needed | harmless | Electron ignores them |
| `main.js` + `preload.js` | ✗ not deployed | ✓ required | Never shipped to web |
| Native menu | ✗ | ✓ | Electron `Menu.buildFromTemplate()` |
| Keyboard shortcuts | both | both | Same handler, registered in `app.js` |
| `storage.js` adapter | both | both | Branches on `window.electronAPI` presence |

### iOS Audio Quirks

The PWA target still needs the `ensureAudio()` async-resume dance for iOS. The Electron target doesn't. Since the detection is simple, handle it in `audio.js`:

```js
// audio.js
async function ensureAudio() {
  if (!audioCtx) audioCtx = new AudioContext();
  // iOS PWA requires resume from a user gesture; Electron doesn't need this
  if (audioCtx.state === 'suspended') await audioCtx.resume();
}
```

This is identical code that happens to be a no-op on Electron. No branching needed — just let it run.

### Deployment

- **PWA:** `npm run build:pwa` → `dist/web/` → push to GitHub Pages branch (same as today, just from the build output instead of the repo root)
- **Electron:** `npm run build:electron` → `dist/electron/Ear Tuner Setup.exe` → distribute manually or via GitHub Releases

### Summary

The dual-target approach adds exactly one abstraction — `storage.js` — and otherwise changes nothing about how you develop. You edit renderer code once, it runs in both places. The `src/renderer/` folder is the app; `src/main/` is the Electron wrapper around it.

---

## Follow-Up: iOS Tags, File Merging, Naming, and Repos

### iOS-Specific Tags — They Stay in index.html

The `<meta name="apple-mobile-web-app-capable">`, `<meta name="apple-mobile-web-app-status-bar-style">`, `<link rel="apple-touch-icon">`, and similar tags all live in `index.html` and stay there permanently. Electron ignores them. Non-Apple browsers ignore them. They're just inert HTML attributes when not on iOS Safari — there's no reason to conditionalize or move them.

The `manifest.json` reference (`<link rel="manifest">`) is the same story: harmless in Electron, required for PWA installability on Android/desktop browsers.

### Does the PWA Build Merge Files Into One index.html?

**No.** The single-file approach of the current app was a convenience for the original "just serve one file" situation. The standard multi-file PWA pattern is:

- `index.html` links to separate CSS and JS files
- The **service worker** caches all of them on first load
- After that, the app works fully offline from the service worker cache

The build step for the PWA does **not** inline or merge files. It:

1. Copies `src/renderer/` → deployment target as-is
2. Generates/injects a cache manifest into `sw.js` (a list of all file paths + a version hash, so the service worker knows which files to cache and when to invalidate)

That's it. No bundling, no inlining.

> [!tip] The source file is still called `index.html`
> The source file at `src/renderer/index.html` is named `index.html` and stays that way. It's in a subfolder of the source repo, not at the root. What gets deployed is a copy of it at the root of the deployment target. No renaming needed at any stage.

### The Repo Problem — and the Right Fix

You've identified the core issue correctly. The current repo (`fiddle-app/ear-tuner`) is serving its root directly via GitHub Pages. That means everything in the repo — backlog, research docs, electron config, `package.json`, `node_modules/` eventually — is publicly reachable. That's wrong.

**The standard practice is to separate source from deployment.** The Windows build output obviously doesn't go into a repo at all (or goes into GitHub Releases as a binary artifact). The web build output goes into a deployment repo (or branch) that contains only what gets served.

### Proposed Repo Split

**Rename the source repo:**

```
fiddle-app/ear-tuner        ← currently exists; rename to:
fiddle-app/ear-tuner-src    ← source repo (private or public, doesn't matter)
```

**Create a new deployment repo:**

```
fiddle-app/ear-tuner        ← new repo, GitHub Pages serves this
                               URL stays: https://fiddle-app.github.io/ear-tuner/
```

The deployment repo contains **only** the output of `npm run build:pwa` — the files that belong in the browser. Nothing else. Its `main` branch is managed entirely by the build script, not by hand.

GitHub will issue a redirect from the old `ear-tuner` URL for a period after the rename, giving you a window to set up the new deployment repo before any links break.

**Naming rationale:** `-src` is unambiguous. `-app` could mean the deployed app. `-dev` implies something temporary. `-src` reads as "this is where the source lives." If this pattern extends to other apps in the family (`tune-hub-src`, `tune-list-src`), it's consistent and easy to understand.

### Updated Folder Structure Across Both Repos

```
fiddle-app/ear-tuner-src/       ← source repo (on disk: fiddle/ear-tuner/)
├── package.json
├── src/
│   ├── main/                   ← Electron main process
│   ├── preload/                ← Electron preload bridge
│   └── renderer/               ← Shared renderer (web + electron)
│       ├── index.html
│       ├── manifest.json
│       ├── sw.js
│       ├── styles/
│       ├── js/
│       └── assets/
├── scripts/
│   ├── build-pwa.js            ← copies renderer/ → ../ear-tuner/ (deployment repo)
│   └── build-electron.js       ← calls electron-builder
├── dist/
│   └── electron/               ← .exe output, gitignored
├── research/
├── backlog.md
├── backlog/
└── CLAUDE.md

fiddle-app/ear-tuner/           ← deployment repo (on disk: some scratch location)
├── index.html                  ← built output only — never hand-edited
├── manifest.json
├── sw.js
├── styles/
│   └── app.css
├── js/
│   ├── app.js
│   └── ...
└── assets/
```

> [!tip] Where does the deployment repo live on disk?
> It doesn't need to be inside your Projects folder at all. The build script clones or pulls it to a temp location, copies the built files in, commits, and pushes. You never open it in an editor. Alternatively, keep it checked out as a sibling: `fiddle/ear-tuner-deploy/` — but don't put it inside `ear-tuner-src/`.

### Build Script Behavior (`build-pwa.js`)

```js
// scripts/build-pwa.js (sketch)
// 1. Copy src/renderer/ to the deployment repo path
// 2. Generate sw-cache-manifest.js with file list + version hash
// 3. cd to deployment repo, git add -A, git commit -m "deploy vX.X", git push
```

The deploy is a single command: `npm run build:pwa`. It produces a commit in `fiddle-app/ear-tuner` with only the web-ready files. GitHub Pages picks it up automatically.

### Summary of Repos After Restructure

| Repo | Purpose | Public | GitHub Pages |
|---|---|---|---|
| `fiddle-app/ear-tuner-src` | Source code, electron config, backlog, research | Optional | No |
| `fiddle-app/ear-tuner` | PWA deployment output only | Yes | Yes → `ear-tuner/` URL |

The `.exe` goes to GitHub Releases on `ear-tuner-src`, not into either repo's working tree.

