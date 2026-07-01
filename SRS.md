# Software Requirements Specification
## MIVA Knowledge Vector Space — v2.0

**Document Type:** SRS (Software Requirements Specification)  
**Version:** 2.0  
**Date:** 2026-06-26  
**Author:** Iyanuoluwa Joseph Akinnusi  
**Status:** Draft — Pre-implementation  

---

## Table of Contents

1. Introduction
2. Overall Description
3. Functional Requirements
4. Non-Functional Requirements
5. Security Requirements
6. UI/UX Requirements
7. Data Requirements
8. Constraints
9. Acceptance Criteria

---

## 1. Introduction

### 1.1 Purpose

This SRS defines the complete requirements for MIVA Knowledge Vector Space v2.0 — an interactive, web-based knowledge visualisation hub for the MIVA university community. It is the source of truth for what the rebuilt `hh.html` must do and how it must behave.

### 1.2 Scope

The system is a **single-file web application** (`hh.html`) that:
- Renders a 3D galaxy metaphor visualising university knowledge as orbiting planets around category suns
- Authenticates MIVA students/staff via Firebase Auth (domain-restricted)
- Reads and writes entries to Firebase Firestore
- Operates in offline/demo mode when Firebase is not configured
- Requires no build toolchain — open in a browser or serve via static hosting

**Out of scope:** Push notifications, offline PWA, admin dashboard, media uploads, real-time collaboration.

### 1.3 Definitions

| Term | Definition |
|---|---|
| **Entry** | A single knowledge node — a piece of information submitted by an authenticated user |
| **Planet** | Visual representation of an Entry in the 3D galaxy |
| **Sun** | Visual representation of a knowledge Category; orbits the black hole |
| **Black Hole** | The MIVA origin point; the gravitational centre of the galaxy |
| **System** | A sun and all its orbiting planets |
| **Focus mode** | State where camera has zoomed into a specific sun's system after a planet click |
| **Ghost mode** | Render state where non-selected systems are drawn at low opacity |

---

## 2. Overall Description

### 2.1 Product Perspective

MIVA Knowledge Vector Space replaces the previous flat 2D force-directed graph (`mivavectorspace.html`) with a fully 3D solar-system galaxy rendered on HTML5 Canvas. The system is stateless from the server's perspective — all rendering happens client-side; Firebase provides auth and data persistence.

### 2.2 User Classes

| Class | Description | Permissions |
|---|---|---|
| **Guest** | Unauthenticated visitor | Read-only: view entries, search, use notepad |
| **Student** | Authenticated @miva.edu.ng user | Guest + submit new entries |
| **Staff** | Authenticated @miva.university user | Same as Student |

No admin role exists in the client. Moderation is handled via Firebase Console.

### 2.3 Operating Environment

- **Browser support:** Chrome 90+, Firefox 88+, Safari 15+, Edge 90+
- **Device support:** Desktop (primary), tablet (functional), mobile (usable but limited)
- **Network:** Works with live Firebase; degrades gracefully to cached data or demo mode
- **Hosting:** Static file — compatible with Vercel, Netlify, GitHub Pages, any HTTP server

---

## 3. Functional Requirements

### FR-01 — 3D Galaxy Rendering

**Priority:** Must Have

The system shall render a real-time 3D galaxy on an HTML5 Canvas using perspective projection:

- **FR-01.1** A central black hole shall be rendered at the origin with an accretion disk, photon ring, and pulsing gravitational lensing waves.
- **FR-01.2** Each knowledge category shall be represented as a sun that orbits the black hole on an inclined elliptical path at a category-specific radius.
- **FR-01.3** Each knowledge entry shall be represented as a planet that orbits its category's sun at a radius determined by the number of entries in that category.
- **FR-01.4** Stars (600 points) shall fill a 3D sphere around the scene and twinkle.
- **FR-01.5** Orbit rings shall be visible for sun orbits (around the black hole) and planet orbits (around each sun).
- **FR-01.6** All scene objects shall be depth-sorted and drawn back-to-front each frame to correctly handle occlusion.
- **FR-01.7** Suns and planets shall spawn with a fade-in animation (staggered by index) after the loading sequence completes.

### FR-02 — Camera & Navigation

**Priority:** Must Have

- **FR-02.1** The user shall be able to drag the canvas to rotate the galaxy (X and Y axis rotation).
- **FR-02.2** The user shall be able to scroll to zoom in and out (zoom range: 0.3× to 4×).
- **FR-02.3** Touch devices shall support single-finger drag to rotate and pinch-to-zoom.
- **FR-02.4** The galaxy shall auto-rotate slowly when no user interaction is occurring.
- **FR-02.5** HUD buttons shall allow: reset camera to default, zoom in, zoom out, toggle auto-rotate.
- **FR-02.6** The camera shall smoothly animate between states (no instant jumps).

### FR-03 — Sun Click & Focus Animation (mobile-first design)

**Priority:** Must Have

**Rationale:** Planets are 5–12 px radius at default zoom — too small to tap reliably on mobile. Suns are 22–30 px and always easy to tap. The solution is a two-step interaction: tap sun → zoom in → tap planet.

- **FR-03.1** When a **sun** is clicked or tapped, the camera shall smoothly animate (600 ms, easeInOutCubic) to centre that sun on screen at approximately 2.8× zoom.
- **FR-03.2** The target camera rotation shall be computed as `ry = atan2(-sun.x, sun.z)` so the sun projects to screen-centre horizontally. The X tilt shall target 0.28 rad.
- **FR-03.3** During the zoom animation and while in focus mode, all orbital angle updates for suns and planets shall be frozen so the focused sun does not drift from centre.
- **FR-03.4** During focus mode, all other category systems (suns + their planets) shall render at ≤10% opacity (ghost mode). The focused sun and its planets remain at full opacity.
- **FR-03.5** The focused sun's orbit rings and planet orbit rings shall render at increased brightness (alpha `28` vs `18`).
- **FR-03.6** After the zoom animation completes (phase = 'hold'), **no panel opens automatically**. The user may now tap individual planets — which are now large enough on screen to tap accurately.
- **FR-03.7** Tapping the **same focused sun again** shall exit focus mode.
- **FR-03.8** Focus mode shall also be exited by: pressing Escape, tapping the "← [Category]" back button, or dragging the canvas.
- **FR-03.9** Exiting focus mode shall animate the camera back to its pre-focus rotation and zoom over 500 ms.

### FR-04 — Hover State

**Priority:** Must Have

- **FR-04.1** Hovering over a planet shall: change cursor to pointer, brighten the planet's glow, and display the planet's full title as a label.
- **FR-04.2** Hovering over a sun shall: change cursor to pointer, display solar flare spikes around the sun, and display the category name.
- **FR-04.3** No tooltip or panel shall open on hover — only on click.

### FR-04b — Planet Click (direct detail)

**Priority:** Must Have

- **FR-04b.1** Tapping a **planet** at any zoom level shall open the detail panel immediately — no secondary zoom animation.
- **FR-04b.2** Closing the detail panel (✕ button) while in focus mode shall close the panel but **keep the camera zoomed in** on the current sun, so the user can tap other planets without losing their zoom context.

### FR-05 — Detail Panel

**Priority:** Must Have

- **FR-05.1** The detail panel shall display: entry title, category badge, subcategory badge, type badge, full content (markdown-rendered), author (obfuscated), date, and tags.
- **FR-05.2** Email addresses and Nigerian phone numbers in content shall be automatically obfuscated (split across DOM nodes to defeat simple regex scrapers).
- **FR-05.3** Markdown formatting shall be supported in content: `**bold**`, `*italic*`, `[label](url)`, and bare HTTPS URLs auto-linked.
- **FR-05.4** All links in the detail panel shall open in a new tab with `rel="noopener noreferrer"`.
- **FR-05.5** All URLs shall be validated — `javascript:`, `data:`, `vbscript:` and other non-HTTP protocols shall be replaced with `#`.

### FR-06 — Search

**Priority:** Must Have

- **FR-06.1** A search input in the header shall filter planets in real time (debounced at 250 ms).
- **FR-06.2** Non-matching planets shall be rendered at ≤8% opacity (very dim) so matches clearly stand out.
- **FR-06.3** Matching planets shall render a **multi-layer search glow**:
  - Layer 1: Diffuse outer radial gradient (4.5× radius, category colour, pulsing slowly)
  - Layer 2: Crisp inner ring (5 px outside body, full category colour, 2 px stroke, pulsing)
  - Layer 3: Softer outer ring (11 px outside body, 66% opacity, offset phase from inner)
  - Label: always shown at full opacity (0.95) for matched planets regardless of zoom level
- **FR-06.4** **Parent suns** of matched planets shall display a secondary glow ring (category colour, 2.5 px stroke, pulsing) so users can identify which sun system contains matches without having to zoom in first.
- **FR-06.5** A match count badge (`N found`) shall appear in the search input.
- **FR-06.6** Clearing the search shall restore all planets and suns to normal rendering.
- **FR-06.7** Search shall match against: title, content, category, subcategory, type, author, and tags.

### FR-07 — Authentication

**Priority:** Must Have

- **FR-07.1** Users shall be able to sign in and sign up with an email/password combination.
- **FR-07.2** Only email addresses ending in `@miva.edu.ng` or `@miva.university` shall be accepted.
- **FR-07.3** The domain restriction shall be enforced both client-side (UX feedback) and server-side (Firestore rules).
- **FR-07.4** The auth modal shall include a password visibility toggle.
- **FR-07.5** A signed-in user's email shall be displayed in the header (truncated to prevent overflow).
- **FR-07.6** A sign-out button shall be available when authenticated.
- **FR-07.7** Firebase App Check (reCAPTCHA v3) shall be active in production to attest requests.
- **FR-07.8** The entry submission button shall only be visible to authenticated users.

### FR-08 — Entry Submission

**Priority:** Must Have

- **FR-08.1** Authenticated users shall be able to submit entries via a left-side panel form.
- **FR-08.2** Required fields: Category, Subcategory (dependent on Category), Type (dependent on Category), Title, Content.
- **FR-08.3** Optional fields: Tags (comma-separated).
- **FR-08.4** Title shall be limited to 150 characters with a live counter.
- **FR-08.5** Content shall be limited to 2000 characters with a live counter.
- **FR-08.6** A live markdown preview shall render below the content textarea.
- **FR-08.7** Toolbar buttons shall insert bold (`**text**`), italic (`*text*`), and link (`[label](url)`) markdown at the cursor.
- **FR-08.8** Link insertion shall validate the URL and reject non-HTTP protocols.
- **FR-08.9** On successful submission, a new planet shall appear in the galaxy immediately (optimistic UI).
- **FR-08.10** Entries shall be saved to Firestore if connected; written to cache if not.
- **FR-08.11** Tags shall be stripped of HTML characters before storage.
- **FR-08.12** In demo mode (Firebase not configured), the form shall show but submission shall be blocked with an explanatory error.

### FR-09 — Notepad

**Priority:** Should Have

- **FR-09.1** A right-side panel shall contain a freeform text notepad.
- **FR-09.2** Notepad content shall be auto-saved to `localStorage` on every keystroke.
- **FR-09.3** A download button shall export the notepad as a `.txt` file.
- **FR-09.4** A clear button shall erase the notepad — requiring a two-click confirmation (no `confirm()` dialog).

### FR-10 — Loading Sequence

**Priority:** Should Have

- **FR-10.1** On load, a hyperspace warp animation shall play (2400 ms) on a separate canvas element.
- **FR-10.2** A loading barrier with progress bar shall be displayed after the warp (1900 ms); the ghost galaxy shall be visible beneath it.
- **FR-10.3** The barrier shall dissolve into the full galaxy (1200 ms fade).
- **FR-10.4** The loading overlay shall be hidden from the DOM after the dissolve completes.
- **FR-10.5** The warp/barrier sequence shall be skipped gracefully on `prefers-reduced-motion`.

### FR-11 — Offline / Demo Mode

**Priority:** Must Have

- **FR-11.1** If Firebase is not configured (placeholder config), the app shall load with sample data.
- **FR-11.2** A visible banner shall inform the user they are in demo mode.
- **FR-11.3** A stale cache (up to 3 hours old) from `localStorage` shall be used if Firestore is unreachable.
- **FR-11.4** Firestore data shall silently revalidate in the background after the cached version is displayed.

### FR-12 — Legend

**Priority:** Must Have

- **FR-12.1** A fixed bottom bar shall show each category's colour dot and icon label.

### FR-13 — Responsive Layout

**Priority:** Should Have

- **FR-13.1** Panels shall be full-width on screens narrower than 768 px.
- **FR-13.2** The search bar shall flex-grow on narrow screens.
- **FR-13.3** The galaxy shall fill the viewport at all screen sizes.
- **FR-13.4** Touch interaction (drag to rotate, pinch to zoom) shall be fully functional on tablets.

---

## 4. Non-Functional Requirements

### NFR-01 — Performance

- The render loop shall sustain ≥30 fps on a mid-range laptop (2020 hardware) with up to 200 planets.
- Physics / orbital updates shall complete within 4 ms per frame.
- Initial data load (from cache) shall complete within 200 ms of page load.
- The zoom animation shall maintain ≥30 fps throughout its 600 ms duration.

### NFR-02 — Reliability

- If Firestore is unavailable, the app shall continue functioning using cached data.
- If the cache is empty and Firestore is unavailable, the app shall load with sample data.
- A failed entry submission shall display an error notification and not corrupt the local state.

### NFR-03 — Maintainability

- All JavaScript shall be contained within a single `<script type="module">` block.
- Functions shall follow the naming convention described in the architectural plan.
- Section comments (`/* ═════ SECTION NAME ═════ */`) shall delineate each logical block.
- No third-party JavaScript libraries except the Firebase SDK (loaded dynamically).

### NFR-04 — Accessibility

- All interactive buttons shall have descriptive `title` attributes.
- Focus shall be visually indicated on all interactive elements (`focus-visible` outline).
- The auth modal shall trap keyboard focus while open.
- The `prefers-reduced-motion` media query shall disable all animations (canvas loop continues but no easing; instant transitions).
- Colour is not the sole means of distinguishing categories — icons are also used.

### NFR-05 — Browser Compatibility

- Shall function without polyfills on Chrome 90+, Firefox 88+, Safari 15+, Edge 90+.
- Dynamic `import()` (ES2020) is required — no fallback for older browsers.
- Canvas 2D API features used: `createRadialGradient`, `arc`, `save`/`restore`, `globalAlpha`, `clip` — all universally supported.

---

## 5. Security Requirements

### SR-01 — Input Sanitization

- **SR-01.1** All user-supplied strings rendered into innerHTML shall pass through the `renderMarkdown()` function which HTML-entity-encodes input before processing.
- **SR-01.2** All URLs inserted into `href` attributes shall pass through `sanitizeUrl()` which blocks `javascript:`, `data:`, and `vbscript:` protocols.
- **SR-01.3** Tags shall have HTML special characters stripped before storage and display.

### SR-02 — External Link Safety

- **SR-02.1** Every anchor tag with `target="_blank"` shall include `rel="noopener noreferrer"`.

### SR-03 — Content Security Policy

- **SR-03.1** The page shall include a meta CSP with at minimum: `frame-ancestors 'none'`, `object-src 'none'`, `base-uri 'self'`, `form-action 'self'`.
- **SR-03.2** The JavaScript frame-buster (`if(self!==top){top.location=self.location}`) shall be retained as a belt-and-suspenders measure.

### SR-04 — Authentication Security

- **SR-04.1** The auth submit button shall be debounced to prevent rapid-fire requests.
- **SR-04.2** Error messages from Firebase Auth shall be displayed to the user but internal error objects shall not be logged to the console in production.
- **SR-04.3** Form submission shall be blocked entirely when `firebaseReady === false`.

### SR-05 — Firebase Credential Management

- **SR-05.1** The production Firebase config shall never be committed to version control.
- **SR-05.2** The working file `hh.html` shall use placeholder strings (`"YOUR_API_KEY"` etc.) for all Firebase config values.
- **SR-05.3** The Firebase API key shall be restricted in Google Cloud Console to the production domain.

### SR-06 — Module Scope

- **SR-06.1** No Firebase SDK methods or internal state shall be attached to `window` or any other global object.
- **SR-06.2** All variables shall remain module-scoped inside `<script type="module">`.

### SR-07 — Firestore Rules

- **SR-07.1** The Firestore security rules shall enforce: public read, authenticated MIVA-domain write only, no client-side update or delete.
- **SR-07.2** The rules shall validate field presence, types, and lengths server-side.
- **SR-07.3** The `author` field shall be validated server-side to match `request.auth.token.email`.

---

## 6. UI/UX Requirements

### UX-01 — Visual Hierarchy

- The black hole is always the most visually dominant element.
- Category suns are secondary — larger than planets, brighter corona.
- Planets are tertiary — varied size based on content length.
- UI panels always feel like they float "above" the galaxy (glassmorphism effect, backdrop blur).

### UX-02 — Interaction Feedback

- Every clickable object shall change the cursor to `pointer`.
- Hover states shall be immediate (no delay).
- The zoom animation shall have a distinct easing curve that feels physical (easeInOutCubic).
- Form submission shall show a loading state on the button while the Firestore write is pending.
- Success and error states shall use toast notifications (top-right, 4 s duration, slide-in/slide-out).

### UX-03 — Colour System

| Token | Value | Usage |
|---|---|---|
| `--bg` | `#030510` | Page background |
| `--bg2` | `#0a0e27` | Card backgrounds |
| `--bg3` | `#0f1535` | Modal background |
| `--primary` | `#E43B31` | MIVA red — primary CTA, danger |
| `--gold` | `#FFB800` | Accretion disk, active HUD icons |
| `--text` | `#ffffff` | Primary text |
| `--text-dim` | `rgba(255,255,255,.55)` | Secondary labels |
| `--glass` | `rgba(10,14,39,.88)` | Panel backgrounds (glassmorphism) |
| `--glass-border` | `rgba(255,255,255,.08)` | Panel borders |

### UX-04 — Panel System

- Only one right panel (Notepad OR Detail) may be open at a time.
- The left panel (Form) may be open simultaneously with a right panel.
- All panels slide in from their respective edges (CSS transform transition, 350 ms, cubic-bezier easing).
- Panels do not overlay the galaxy canvas on desktop — they sit on top with blur backdrop.

### UX-05 — Dismissible UI

- The info tip bar shall be dismissible and remember the dismiss state via `localStorage`.
- The Firebase demo mode banner shall be dismissible per session.

### UX-06 — Empty State

- If no entries exist (empty Firestore + no cache + no sample data matches), the black hole and suns shall still render. A subtle text overlay shall read: "No knowledge entries yet. Sign in to add the first."

---

## 7. Data Requirements

### DR-01 — Entry Schema

```
Entry {
  id:          string    — Firestore document ID (auto-generated)
  title:       string    — 1..150 chars
  content:     string    — 1..2000 chars
  type:        string    — enum: see ETYPES constant
  category:    string    — enum: Faculty | Contacts | Exams | Projects | Resources | News | Ideas
  subcategory: string    — enum: see SUBCATS constant
  author:      string    — Firebase Auth email
  createdAt:   string    — ISO date YYYY-MM-DD
  tags:        string[]  — 0..10 items, each 1..30 chars, no HTML
}
```

### DR-02 — localStorage Schema

```
miva_knowledge_cache_v2   → { data: Entry[], timestamp: number }   3-hour TTL
miva_notepad_content      → string (plain text)
miva_info_dismissed       → '1' (flag)
```

### DR-03 — Data Integrity

- The system shall not render an entry with a missing `category` — it shall be silently skipped.
- The system shall not render an entry with a missing `title` — the planet shall display `[Untitled]`.
- Corrupt or non-JSON data in localStorage shall be silently caught and the cache discarded.

---

## 8. Constraints

| Constraint | Detail |
|---|---|
| Single file | All HTML, CSS, and JS in one `.html` file. No build step. |
| No frameworks | Vanilla JS, CSS, HTML only. Firebase SDK loaded dynamically via ES module imports. |
| No server-side code | All logic runs in the browser. Firebase is the backend. |
| No media uploads | Entries are text-only. No image or file attachments. |
| No edit/delete for users | Once published, entries are immutable from the client. Corrections go through admin. |
| Font dependency | Inter from Google Fonts (or self-hosted). Fallback: system-ui. |
| Module requirement | `<script type="module">` required for ES dynamic import. No IE11 support. |

---

## 9. Acceptance Criteria

### AC-01 — Galaxy renders correctly
- [ ] Black hole visible at scene centre with accretion disk and pulsing lensing rings
- [ ] All 7 category suns orbit the black hole at different radii with visible orbit rings
- [ ] Knowledge entries appear as planets orbiting their parent sun
- [ ] Stars fill the background; galaxy auto-rotates on idle

### AC-02 — Sun click zoom animation
- [ ] Clicking/tapping a sun triggers a smooth 600 ms easeInOutCubic zoom to 2.8× with that sun centred
- [ ] Orbital motion freezes while focused so the sun does not drift from centre
- [ ] All other systems fade to ≤10% opacity during focus mode
- [ ] No panel opens automatically — user taps planets individually
- [ ] Tapping the same sun again, pressing Escape, or pressing "← Back" smoothly returns camera to pre-click state over 500 ms
- [ ] Closing the detail panel (✕) stays zoomed — does not exit focus

### AC-02b — Planet tap at zoomed level
- [ ] After zooming in on a sun, tapping a planet opens the detail panel directly
- [ ] Planet tap at any zoom level (including default) opens detail directly with no secondary zoom

### AC-03 — Detail panel content
- [ ] Title, category, subcategory, type displayed correctly
- [ ] Markdown bold, italic, and links render correctly
- [ ] Email addresses are obfuscated with `[at]` / `[dot]` DOM splitting
- [ ] All links have `rel="noopener noreferrer"`
- [ ] `javascript:` URLs are replaced with `#`

### AC-04 — Authentication
- [ ] Sign in works with `@miva.edu.ng` email
- [ ] Sign in is rejected for non-MIVA emails with a clear error
- [ ] Entry form appears only when authenticated
- [ ] Sign out clears the user state and hides the form

### AC-05 — Entry submission
- [ ] Form validates all required fields before submission
- [ ] Successful submission adds a new planet to the galaxy immediately
- [ ] Submission is blocked in demo mode with an explanatory error

### AC-06 — Search glow
- [ ] Typing in search field fades non-matching planets to ≤8% opacity within 250 ms
- [ ] Matching planets show three-layer glow (diffuse radial, crisp ring, outer soft ring) all pulsing
- [ ] Matched planet labels are always visible at full opacity regardless of zoom
- [ ] Parent suns of matching planets show a secondary glow ring
- [ ] Match count badge shows in search field
- [ ] Clearing search restores all planets and suns to normal rendering

### AC-07 — Security
- [ ] Page source does not contain live Firebase credentials
- [ ] `javascript:alert(1)` in a markdown link renders as `href="#"`, not an executable link
- [ ] `<script>alert(1)</script>` in entry content renders as escaped text, not executed
- [ ] Auth email field in the header uses `textContent`, not `innerHTML`
- [ ] `window._fb` does not exist in the browser console
- [ ] CSP meta tag includes `object-src 'none'` and `base-uri 'self'`

### AC-08 — Performance
- [ ] Galaxy renders at ≥30 fps on a 2020 mid-range laptop with 50+ planets
- [ ] Zoom animation is visually smooth (no jank/stutter) on the same hardware

### AC-09 — Demo mode
- [ ] App loads with sample data when Firebase config is set to `"YOUR_*"` placeholders
- [ ] Demo mode banner is visible and dismissible
- [ ] Form shows but submission is blocked with an error in demo mode
