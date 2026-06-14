# MIVA Knowledge Vector Space 🌌

MIVA Knowledge Vector Space is a premium, single-file, interactive knowledge visualization hub designed for university communities. It maps and clusters information (courses, contacts, exams, department news, projects, resources, and ideas) onto a dynamic, force-directed graph rendered on an HTML5 canvas.

---

## 🚀 Key Features

*   **Interactive Force-Directed Graph:** Smooth physics-based clustering, nodes repelling each other while being attracted to category centers.
*   **Aesthetics & Cosmic Theme:** Sleek dark cosmic design with responsive canvas scaling, stars twinkling, glassmorphism panels, and vibrant, custom color palettes.
*   **Offline/Demo Mode Fallback:** Automatically switches to demo mode using pre-packaged university records if Firebase is not configured.
*   **Dual Panel System:** 
    *   *Left Panel:* Submission portal with category-subcategory dependent dropdown validation.
    *   *Right Panel:* Auto-saved notepad utilizing `localStorage` with a local `.txt` file downloader.
*   **Debounced Search:** Instant search filter highlighting matching entries with a visual pulsing effect.
*   **Anti-Scraper Defenses:**
    *   *Phone/Email Obfuscation:* Dynamically replaces contact info pattern sequences in the DOM with spaced elements (`REDACTED`) to bypass bot regular expressions.
    *   *Frame Buster:* Direct CSP directives and JavaScript check scripts preventing clickjacking or unauthorized `<iframe>` embed rendering.

---

## 🛠 Tech Stack

*   **Frontend:** Vanilla HTML5, CSS3, ES6 Modules.
*   **Canvas Render Engine:** Native HTML5 Canvas 2D Context.
*   **Database & Auth:** Firebase v10 Auth (Email/Password with domain validation) & Cloud Firestore.
*   **Attestation Security:** Firebase App Check & Google reCAPTCHA v3.

---

## 📂 Project Structure

```
Miva Vector Space/
│
├── INDEX.html    # Standalone application containing all inline CSS/JS and logo assets
├── sample-structure.json     # Sample dataset mapping nodes and linkages
└── README.md                 # Complete project guide and setup instructions
```

---

## 🔒 Firestore Security Rules

To ensure guest users can read entries securely without registering, but restrict writing strictly to authenticated university members, apply the following rules in your **Firebase Console** under **Firestore Database** > **Rules**:

```firestore
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    
    
    // MIVA Vector Space Rules
    match /miva_entries/{entryId} {
      // Anyone (guests/public) can view the space entries
      allow read: if true;
      
      // Only signed-in users can add new entries
      allow create: if request.auth != null;
      
      // Prevent anyone (even authenticated users) from editing or deleting entries
      allow update, delete: if false;
    }
  }
}
```

---

## 🛡 Enabling Anti-Scraping Attestation (App Check)

To prevent scrapers from querying your Firestore API endpoints directly from outside your domain, set up **Firebase App Check**:

1.  **Register reCAPTCHA v3:** Go to [Google reCAPTCHA Console](https://www.google.com/recaptcha/admin/). Create keys using **reCAPTCHA v3**. Add your production domains (e.g. Vercel).
2.  **Enable App Check in Firebase:** In Firebase Console > Build > App Check, register **reCAPTCHA v3** using your *Secret Key*. Go to the **APIs** tab, choose **Cloud Firestore**, and click **Enforce**.
3.  **Include App Check in HTML:**
    Insert the following script under the firebase initialization inside `index.html`:
    ```javascript
    const { initializeAppCheck, ReCaptchaV3Provider } = await import('https://www.gstatic.com/firebasejs/10.12.0/firebase-app-check.js');
    const appCheck = initializeAppCheck(firebaseApp, {
      provider: new ReCaptchaV3Provider('YOUR_RECAPTCHA_SITE_KEY'), // Add your public Site Key here
      isTokenAutoRefreshEnabled: true
    });
    ```

---

## 💻 Running Locally

Due to browser security policies surrounding ES Modules (`import` statements), you cannot open the HTML file directly (`file://` protocol) in a browser. Run a local web server:

### Python Method (Recommended)
Open your terminal in the directory and run:
```bash
python -m http.server 8005
```
Open **`http://localhost:8005/index.html`** in your browser.

### Node.js Method
```bash
npx serve .
```

---

## 🌐 Deploying to Production (Vercel)

1.  Initialize a Git repository and commit your files:
    ```bash
    git init
    git add .
    git commit -m "Initial commit"
    ```
2.  Push to GitHub, GitLab, or Bitbucket.
3.  Link your repository in your [Vercel Dashboard](https://vercel.com).
4.  Vercel will auto-detect the workspace. Choose **Other** as the framework template and deploy.
5.  Link the deployment domain inside your Google reCAPTCHA domains list.
