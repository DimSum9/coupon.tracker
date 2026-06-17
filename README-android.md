# Coupon Vault — Android Build Guide

This folder contains everything needed to publish Coupon Vault as a real Android app using a **Trusted Web Activity (TWA)**. The app runs your existing HTML/JS code natively inside a Chrome-engine shell — no rewrite needed, full Play Store support.

---

## How it works

A **TWA (Trusted Web Activity)** is the official Google-recommended way to ship a web app as a real Android APK. It:
- Opens your hosted website fullscreen with **no browser chrome** (no address bar, no tabs)
- Has its own launcher icon, splash screen, and app name
- Passes Play Store review (used by Pinterest, Starbucks, Twitter Lite, and hundreds more)
- Supports push notifications, camera access, offline use (via Service Worker)
- Is **not** a WebView hack — it uses the real Chrome engine

---

## Prerequisites

Install these once on your machine:

| Tool | Install |
|------|---------|
| Node.js 18+ | https://nodejs.org |
| Java JDK 17+ | https://adoptium.net |
| Android Studio (for SDK) | https://developer.android.com/studio |
| Bubblewrap CLI | `npm install -g @bubblewrap/cli` |

---

## Step 1 — Host your web app

The TWA needs your app served over **HTTPS** from a real domain. Options:

### Option A — GitHub Pages (free, fastest)
```bash
# Create a new GitHub repo, push these files:
git init
git add index.html manifest.json service-worker.js icons/ .well-known/
git commit -m "Coupon Vault PWA"
git remote add origin https://github.com/YOUR_USERNAME/coupon-vault.git
git push -u origin main

# In GitHub repo Settings → Pages → Source: main branch, root /
# Your app will be live at: https://YOUR_USERNAME.github.io/coupon-vault/
```

### Option B — Netlify (free, custom domain)
```bash
npm install -g netlify-cli
netlify deploy --prod --dir .
# Follow prompts — get a URL like https://coupon-vault-abc123.netlify.app
```

### Option C — Firebase Hosting (free tier)
```bash
npm install -g firebase-tools
firebase login
firebase init hosting
firebase deploy
```

**Important:** After deploying, note your domain (e.g. `your-username.github.io`). You'll need it in Step 2.

---

## Step 2 — Configure your domain

Edit `twa-manifest.json` and replace every `your-domain.com` with your actual domain:

```json
{
  "host": "your-username.github.io",
  "startUrl": "/coupon-vault/index.html",
  "fullScopeUrl": "https://your-username.github.io/coupon-vault/"
}
```

Also update `app/build.gradle`:
```gradle
manifestPlaceholders = [
    hostName: "your-username.github.io",
    defaultUrl: "https://your-username.github.io/coupon-vault/index.html",
    ...
]
```

---

## Step 3 — Generate a signing keystore

Every Android app must be signed. Run this once and **keep the keystore file safe**:

```bash
keytool -genkey -v \
  -keystore android.keystore \
  -alias couponvault \
  -keyalg RSA \
  -keysize 2048 \
  -validity 10000 \
  -dname "CN=Coupon Vault, OU=Mobile, O=YourName, L=City, S=State, C=US"

# You'll be prompted for a password — remember it
```

Get your SHA-256 fingerprint (needed for asset linking):
```bash
keytool -list -v -keystore android.keystore -alias couponvault | grep SHA256
# Output looks like: AB:CD:EF:12:34:...
```

---

## Step 4 — Set up Digital Asset Links

This is what proves to Android that your app "owns" the website (prevents address bar showing).

1. Copy the SHA-256 fingerprint from Step 3
2. Edit `.well-known/assetlinks.json`:
```json
[{
  "relation": ["delegate_permission/common.handle_all_urls"],
  "target": {
    "namespace": "android_app",
    "package_name": "com.couponvault.app",
    "sha256_cert_fingerprints": [
      "AB:CD:EF:12:34:56:78:90:AB:CD:EF:12:34:56:78:90:AB:CD:EF:12:34:56:78:90:AB:CD:EF:12:34:56:78"
    ]
  }
}]
```
3. Deploy this file so it's accessible at:
   `https://your-domain.com/.well-known/assetlinks.json`

Verify it works:
```
https://digitalassetlinks.googleapis.com/v1/statements:list?source.web.site=https://your-domain.com&relation=delegate_permission/common.handle_all_urls
```

---

## Step 5 — Build the APK

### Option A — Using Bubblewrap (recommended)

```bash
# Initialize (reads twa-manifest.json)
bubblewrap init --manifest https://your-domain.com/manifest.json

# Build debug APK for testing
bubblewrap build

# Sign the release APK
bubblewrap build --release
```

The APK will be at: `./app-release-signed.apk`

### Option B — Using Android Studio

1. Open Android Studio → **Open** → select this folder
2. Wait for Gradle sync to complete
3. Set your keystore passwords in `local.properties`:
   ```
   KEYSTORE_PASSWORD=your_password
   KEY_PASSWORD=your_password
   ```
4. **Build** → **Generate Signed Bundle/APK** → APK → select `android.keystore`
5. Choose `release` build variant → Finish

---

## Step 6 — Test on device

```bash
# Install debug APK directly to connected Android device
adb install app-debug.apk

# Or drag the APK file onto an Android emulator
```

The app should open fullscreen with no browser chrome. If you see an address bar, the asset links verification hasn't propagated yet — wait a few hours and try again.

---

## Step 7 — Publish to Play Store

1. Go to https://play.google.com/console
2. Create a new app → **Create app**
3. Fill in store listing (name, description, screenshots)
4. Upload your signed `app-release-signed.apk` or build an AAB:
   ```bash
   bubblewrap build --release --format aab
   ```
5. Complete content rating questionnaire
6. Submit for review (usually 1–3 days)

---

## Replace placeholder icons

The current `icons/icon-192.png` and `icon-512.png` are solid green placeholders.
To replace them with the real vault logo from `icons/icon.svg`:

```bash
# Using Inkscape (free):
inkscape icons/icon.svg -w 192 -h 192 -o icons/icon-192.png
inkscape icons/icon.svg -w 512 -h 512 -o icons/icon-512.png

# Using ImageMagick:
convert -background none icons/icon.svg -resize 192x192 icons/icon-192.png
convert -background none icons/icon.svg -resize 512x512 icons/icon-512.png

# Online: https://svgtopng.com — upload icon.svg, export at 192 and 512
```

Then regenerate the Android mipmap density icons:
```bash
convert icons/icon-512.png -resize 48x48   app/src/main/res/mipmap-mdpi/ic_launcher.png
convert icons/icon-512.png -resize 72x72   app/src/main/res/mipmap-hdpi/ic_launcher.png
convert icons/icon-512.png -resize 96x96   app/src/main/res/mipmap-xhdpi/ic_launcher.png
convert icons/icon-512.png -resize 144x144 app/src/main/res/mipmap-xxhdpi/ic_launcher.png
convert icons/icon-512.png -resize 192x192 app/src/main/res/mipmap-xxxhdpi/ic_launcher.png
# Copy each to the matching ic_launcher_round.png as well
```

---

## Project file map

```
coupon-vault-android/
├── index.html                          ← Your web app (rename from coupon-vault.html)
├── manifest.json                       ← PWA web manifest
├── service-worker.js                   ← Offline caching + push notifications
├── twa-manifest.json                   ← Bubblewrap config
├── build.gradle                        ← Root Gradle build
├── settings.gradle                     ← Gradle project settings
├── android.keystore                    ← Your signing key (generate in Step 3)
├── icons/
│   ├── icon.svg                        ← Master icon (source of truth)
│   ├── icon-192.png                    ← PWA icon (replace placeholder)
│   └── icon-512.png                    ← PWA icon large (replace placeholder)
├── .well-known/
│   └── assetlinks.json                 ← Digital Asset Links (deploy to your server)
└── app/
    ├── build.gradle                    ← App module Gradle config
    └── src/main/
        ├── AndroidManifest.xml         ← Android permissions + activity config
        └── res/
            ├── values/
            │   ├── strings.xml
            │   ├── colors.xml
            │   └── styles.xml
            ├── drawable/splash.xml     ← Splash screen
            ├── xml/file_paths.xml      ← FileProvider paths (for camera)
            └── mipmap-*/
                └── ic_launcher*.png    ← App icons at each density
```

---

## Changing the package name / app ID

The default package name is `com.couponvault.app`. To change it:

1. `app/build.gradle` → `applicationId`
2. `AndroidManifest.xml` → `android:authorities` in FileProvider
3. `twa-manifest.json` → `packageId`
4. `.well-known/assetlinks.json` → `package_name`
5. Re-run keytool and update SHA-256

---

## Common issues

| Problem | Fix |
|---------|-----|
| Address bar shows in app | Asset links not verified — check `.well-known/assetlinks.json` is publicly accessible and SHA-256 matches |
| App crashes on launch | Domain in `build.gradle` doesn't match deployed URL |
| Camera not working | Add `<uses-permission android:name="android.permission.CAMERA"/>` (already included) |
| Service worker not installing | Must be served over HTTPS, not `file://` |
| Build fails with SDK error | Open SDK Manager in Android Studio, install API 34 |
