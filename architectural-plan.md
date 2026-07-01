# MIVA Knowledge Vector Space — Architectural Plan

**Version:** 2.0 (Full Rebuild)  
**Date:** 2026-06-26  
**Working File:** `hh.html`  
**Status:** Pre-implementation planning

---

## 1. Overview

The rebuild consolidates the 3D galaxy approach in `hh.html` (superior to the 2D force-directed graph in `mivavectorspace.html`) into a hardened, production-quality single HTML file. The core metaphor is a **solar system galaxy**: a central MIVA black hole, category suns orbiting it, and knowledge-entry planets orbiting each sun.

### 1.1 Design Decisions

| Decision      | Choice                                   | Rationale                              |
| ------------- | ---------------------------------------- | -------------------------------------- |
| Rendering     | HTML5 Canvas 2D with 3D perspective math | No dependency, runs everywhere         |
| Backend       | Firebase Auth + Firestore                | Already integrated, proven stack       |
| Distribution  | Single HTML file                         | Deployable as static file anywhere     |
| Auth model    | Domain-restricted email/password         | Only @miva.edu.ng and @miva.university |
| Vizualisation | 3D solar system (hh.html approach)       | More immersive than flat graph         |

---

## 2. Visual Architecture

### 2.1 Galaxy Hierarchy

```
BLACK HOLE (MIVA origin — centre of scene)
│
├── SUN: Faculty  ──orbits BH at radius 280──  7 category suns total
│     └── PLANET: CS101 Syllabus
│     └── PLANET: MTH201 Course
│     └── ...
│
├── SUN: Contacts  ──orbits BH at radius 340──
│     └── PLANET: CS Dept Rep
│     └── ...
│
└── ...
```

### 2.2 Scene Layers (depth-sorted, back to front)

1. Deep-space star sphere (3D, 600 stars, no interaction)
2. Galactic plane orbit rings (faint, decorative)
3. Category sun orbit rings (per-sun, coloured)
4. Black hole (accretion disk, photon ring, gravitational lensing pulses)
5. Suns (corona + solid core, solar flare on hover)
6. Planet orbit rings (per-sun, per-layer)
7. Planets (glow + body + highlight + label on hover)
8. **Focus overlay** (new — dims non-selected system on zoom)

### 2.3 Camera Model

```
State: {
  rot:  { x, y }        — Euler angles (X = tilt, Y = spin)
  zoom: number           — scalar applied to perspective distance
  focus: null | { sunX, sunY, sunZ, targetZoom } — planet click zoom state
  focusProgress: 0..1   — animation t value
  autoRot: boolean
}
```

### 2.4 Sun Click — Zoom & Focus Animation

**Interaction model (mobile-first):**

- Suns are large and always easy to tap (22–30 px radius × scale).
- Planets are small (5–12 px). On mobile at default zoom they are too small to tap reliably.
- **Solution**: tap a sun → zoom in to 2.8× centred on that sun → planets are now large enough to tap → tap a planet → detail panel.

**Zoom-in sequence:**

1. **Record** `S.preFocus = {rot, zoom}` before animating.
2. **Compute target rotation**: `targetRy = atan2(-sun.x, sun.z)` — brings the sun to screen-centre (solves `x·cos(ry) + z·sin(ry) = 0`). Use shortest arc delta.
3. **Animate** over 600 ms with `easeInOutCubic`:
   - `S.rot.y` → `targetRy`
   - `S.rot.x` → `0.28 rad` (comfortable top-down view)
   - `S.cam.zoom` → `2.8`
4. **Orbital freeze**: while `S.focus` is set, sun/planet angles are not incremented (scene freezes so planets don't drift from the centred sun).
5. **During focus**: all other category systems render at ≤10% alpha (ghost mode). The focused sun and its planets remain at full alpha.
6. **After animation** (`phase = 'hold'`): no auto-panel — user taps individual planets (which are now large enough on screen) to open the detail panel.
7. **Exit focus**: tap the focused sun again, press Escape, tap "← [Category]" back button, or drag the canvas. Reverse animation restores `preFocus.rot` and `preFocus.zoom` over 500 ms.

```
// Easing
function easeInOutCubic(t) {
  return t < 0.5 ? 4*t*t*t : 1 - Math.pow(-2*t + 2, 3) / 2;
}

// Target Y angle to centre a sun at screen-centre
const targetRy = Math.atan2(-sun.x, sun.z);
```

**Tap on planet (at any zoom level)**: opens detail panel directly. No secondary zoom animation.

---

## 3. Loading Sequence

```
PHASE_WARP    (2400 ms)  — Hyperspace warp tunnel on warp-canvas
PHASE_BARRIER (1900 ms)  — Loading barrier with progress bar; ghost galaxy visible underneath
PHASE_DISSOLVE (1200 ms) — Barrier dissolves, full galaxy fades in
PHASE_GALAXY  (∞)        — Interactive mode
```

The loading sequence lives in `hh.html` and is preserved as-is (it's already excellent).

---

## 4. Component Map

```
hh.html
├── <head>
│   ├── CSP meta (frame-ancestors + script-src + style-src)
│   ├── Google Fonts (Inter) with SRI hash
│   └── <style> — All CSS, CSS variables, animations
│
├── <body>
│   ├── #app
│   │   ├── canvas#main-canvas         — Galaxy renderer
│   │   ├── canvas#warp-canvas         — Hyperspace effect
│   │   ├── #loading-overlay           — Full warp/barrier sequence
│   │   ├── .header                    — Logo, search, auth buttons
│   │   ├── #toggle-form (left btn)    — Show/hide submit panel
│   │   ├── #toggle-notepad (right btn)— Show/hide notepad
│   │   ├── .panel#panel-form          — Entry submission
│   │   ├── .panel#panel-notepad       — Auto-saving notepad
│   │   ├── .panel#panel-detail        — Planet detail view (slides in after zoom)
│   │   ├── #auth-modal                — Sign-in / sign-up modal
│   │   ├── #notif-container           — Toast notifications
│   │   ├── .legend                    — Category colour key
│   │   ├── .info-tip                  — Dismiss-once help bar
│   │   └── .hud-controls              — Zoom, reset, rotate buttons
│   │
│   └── <script type="module">
│       ├── Firebase init              — Conditional (only if config set)
│       ├── CONSTANTS                  — CATEGORIES, SUBCATS, ETYPES
│       ├── SAMPLE                     — Demo data
│       ├── STATE (S)                  — Single source of truth
│       ├── 3D math (project)
│       ├── Stars (initStars)
│       ├── Galaxy builder (buildGalaxy)
│       ├── Draw helpers               — drawOrbitRing, drawBlackHole, drawSun, drawPlanet
│       ├── Focus animation            — zoomToSun(), zoomOut()
│       ├── Render loop (loop)
│       ├── Hit test (hitTest)
│       ├── Canvas events              — mouse, touch
│       ├── HUD buttons
│       ├── Content helpers            — sanitizeUrl, obfuscateContent, renderMarkdown
│       ├── showDetail / closeDetail
│       ├── Search (doSearch)
│       ├── Panel toggles
│       ├── Notepad
│       ├── Auth (modal, sign-in, sign-up, state change)
│       ├── Form (submit entry)
│       ├── Cache                      — localStorage with TTL
│       ├── Data loading (loadData)
│       ├── Legend
│       └── Init                       — resize, initStars, loadData, loop
```

---

## 5. Security Architecture

### 5.1 Layered Security Model

```
Layer 1 — Browser/Transport
  ├── HTTPS (enforced at deploy)
  ├── CSP: frame-ancestors 'none'  (clickjacking)
  ├── CSP: script-src 'self' www.gstatic.com (no arbitrary script injection)
  ├── CSP: style-src 'self' fonts.googleapis.com 'unsafe-inline'
  └── JS frame-buster (belt-and-suspenders with CSP)

Layer 2 — Content Security
  ├── sanitizeUrl() — strips javascript:/data:/vbscript: from all hrefs
  ├── renderMarkdown() — HTML entity-encodes input BEFORE any regex
  ├── rel="noopener noreferrer" on all external links
  └── No innerHTML with raw user strings (use textContent or sanitized output)

Layer 3 — Authentication
  ├── Firebase Auth (email/password)
  ├── Domain whitelist: @miva.edu.ng or @miva.university
  ├── reCAPTCHA v3 via Firebase App Check
  └── Session managed by Firebase SDK (not localStorage)

Layer 4 — Data
  ├── Firestore rules: read=public, write=auth only, update/delete=false
  ├── Client-side field validation (length, required fields, format)
  ├── No eval(), no Function(), no new Function()
  └── Tag sanitization: strip HTML from tags before storage

Layer 5 — Anti-scraping
  ├── Email obfuscation in rendered content (DOM split)
  ├── Phone obfuscation (Nigerian format)
  ├── Firebase App Check enforcement on Firestore API
  └── Rate-limiting delegated to Firebase Auth
```

### 5.2 CSP Header (meta tag)

```html
<meta
  http-equiv="Content-Security-Policy"
  content="
  default-src 'self';
  script-src 'self' https://www.gstatic.com https://www.google.com https://www.grecaptcha.net 'nonce-RUNTIME_NONCE';
  style-src 'self' https://fonts.googleapis.com 'unsafe-inline';
  font-src 'self' https://fonts.gstatic.com;
  connect-src 'self' https://*.googleapis.com https://*.firebaseio.com https://firestore.googleapis.com;
  img-src 'self' data: https:;
  frame-ancestors 'none';
  base-uri 'self';
  form-action 'self';
  object-src 'none';
"
/>
```

> **Note:** Because Firebase SDK is loaded dynamically via `import()`, `script-src` must include `https://www.gstatic.com`. The `nonce` approach is preferable for inline scripts if server-side rendering is introduced later; for pure static hosting, the `type="module"` tag must be allowed — add `'unsafe-inline'` to script-src only if needed by the module approach, and document why.

---

## 6. State Management

```javascript
const S = {
  // Phase control
  phase: PHASE_WARP,
  phaseStart: Date.now(),

  // Camera
  rot: { x: 0.46, y: 0 },
  autoRot: true,
  autoRotSpeed: 0.0008,
  cam: { fov: 900, zoom: 1 },

  // Focus animation (NEW)
  focus: null, // { planet, sun, fromRot, fromZoom, startTime }
  focusProgress: 0,
  focusDuration: 600, // ms

  // Interaction
  drag: null,
  hover: null,
  selected: null,

  // Data
  entries: [],
  suns: [],
  planets: [],
  stars: [],

  // Render
  bhPulse: 0,
  frame: 0,
  dissolveAlpha: 1,

  // Search
  query: "",
  searchTimer: null,

  // Auth
  user: null,
};
```

---

## 7. Key Function Signatures (Contracts)

```javascript
// 3D projection
function project(x, y, z) → { sx, sy, scale, depth }

// URL security (NEW — replaces inline href usage)
function sanitizeUrl(rawUrl) → string (safe href or '#')

// Markdown renderer (security-hardened)
function renderMarkdown(text) → HTML string (all hrefs sanitized)

// Planet focus animation (NEW)
function zoomToSun(sun) → void        // sets S.focus, triggers 600 ms zoom centred on sun
function zoomOut() → void              // reverse-animates back to prior state

// In loop():
function tickFocusAnimation() → void  // called every frame; updates S.rot/S.cam.zoom from S.focus

// Data
function buildGalaxy() → void          // rebuilds S.suns and S.planets from S.entries
async function loadData() → void       // loads from cache then Firestore

// Form
function validateEntry(fields) → { valid: boolean, errors: string[] }
```

---

## 8. Data Model

### 8.1 Firestore Collection: `miva_entries`

```
{
  id:          string    (Firestore auto-id)
  title:       string    max 150 chars
  content:     string    max 2000 chars
  type:        string    enum from ETYPES[category]
  category:    string    enum from CAT_KEYS
  subcategory: string    enum from SUBCATS[category]
  author:      string    Firebase Auth uid or email (obfuscated on render)
  createdAt:   string    ISO date YYYY-MM-DD
  tags:        string[]  max 10 tags, each max 30 chars, no HTML
}
```

### 8.2 localStorage Keys

```
miva_knowledge_cache_v2  — { data: Entry[], timestamp: number } — 3h TTL
miva_notepad_content     — string — plain text
miva_info_dismissed      — '1'   — one-time dismiss flag
```

---

## 9. File Structure (single file, organised sections)

```
hh.html
│
Section A: HEAD — meta, CSP, fonts, favicon
Section B: CSS  — variables, animations, layout, components
Section C: HTML — app shell, panels, modals, controls
Section D: JS   — (type="module")
  │
  D1  Firebase initialisation
  D2  Constants (CATEGORIES, SUBCATS, ETYPES, SAMPLE, PHASES)
  D3  State (S)
  D4  DOM helpers ($, $$)
  D5  3D math (project, easeInOutCubic)
  D6  Stars
  D7  Galaxy builder
  D8  Draw: orbit rings
  D9  Draw: black hole
  D10 Draw: sun
  D11 Draw: planet
  D12 Draw: focus overlay (NEW)
  D13 Update orbitals
  D14 Galaxy render (renderGalaxy)
  D15 Warp renderer
  D16 Focus animation (zoomToPlanet, zoomOut, tickFocusAnimation) (NEW)
  D17 Main loop
  D18 Hit test
  D19 Canvas events (mouse, touch, keyboard)
  D20 HUD buttons
  D21 Content helpers (sanitizeUrl, obfuscateContent, renderMarkdown) (HARDENED)
  D22 showDetail / closeDetail
  D23 Search
  D24 Panels
  D25 Notepad
  D26 Notifications
  D27 Auth
  D28 Form (validateEntry, submit handler)
  D29 Cache
  D30 Data loading
  D31 Legend
  D32 Info tip
  D33 Firebase banner
  D34 Resize
  D35 Init
```

---

## 10. Enhancements Over Current hh.html

| Feature                  | Current                       | Rebuilt                                                   |
| ------------------------ | ----------------------------- | --------------------------------------------------------- |
| Sun click                | No action                     | Zoom animation (600 ms) → sun centred at 2.8×             |
| Planet click             | Shows panel immediately       | Shows panel immediately (no zoom)                         |
| Focus state              | None                          | Other systems dim to 10% alpha                            |
| Sun centred during zoom  | N/A                           | Yes — atan2(-x,z) rotation target                         |
| Mobile tap accuracy      | Planets too small             | Zoom in on sun first, then planets are accessible         |
| URL sanitization         | None                          | `sanitizeUrl()` blocks `javascript:`                      |
| External link safety     | No `rel` attr                 | `rel="noopener noreferrer"` on all                        |
| CSP                      | `frame-ancestors 'none'` only | Full CSP including `script-src`, `object-src`, `base-uri` |
| innerHTML with user data | Yes (XSS risk)                | Sanitized output only                                     |
| `window._fb` global      | Yes (leaks SDK)               | Removed; module-scoped                                    |
| Keyboard support         | None                          | `Escape` = zoom out / close panel                         |
| Focus ring on keyboard   | None                          | `focus-visible` outlines on buttons                       |
| Reduced motion           | Media query present           | Fully honoured (no animation)                             |
| Entry tag sanitization   | None                          | Strip HTML from tags                                      |
| Demo mode email          | Any email accepted            | Domain check applies in demo mode too                     |
| SRI on fonts             | None                          | SRI hash on Google Fonts link                             |

---

## 11. Deployment Checklist

- [ ] Firebase project configured with correct domain allowlist
- [ ] Firestore security rules deployed (read=public, write=auth, no update/delete)
- [ ] App Check enabled and reCAPTCHA v3 site key set
- [ ] Production domain added to reCAPTCHA console
- [ ] Firebase API key restricted to HTTP referrer (in Google Cloud Console)
- [ ] HTTPS enforced at host (Vercel, Netlify, etc.)
- [ ] Google Fonts SRI hash verified
- [ ] Firebase config keys replaced in hh.html
- [ ] reCAPTCHA key replaced in hh.html
- [ ] `console.warn` / `console.log` statements reviewed or removed
- [ ] Test in-scope Firestore rules with Firebase Rules Playground
