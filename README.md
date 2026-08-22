# TechAurix Command Deck — APK build

This repo wraps `techaurix-app.html` (the arm/cam/drive command deck) in a
minimal Android WebView app and builds it into an installable `.apk` using
GitHub Actions — no Android Studio and no PC required.

```
techaurix-android/
├── index.html                  ← same web app, for optional GitHub Pages hosting
├── android/                    ← the Android WebView wrapper project
│   └── app/src/main/assets/techaurix-app.html   ← the app bundled offline
└── .github/workflows/build-apk.yml              ← builds the APK on every push
```

## 1. Push this to GitHub (from Chromebook/phone, no git needed)

1. Go to https://github.com/new and create a repo, e.g. `techaurix-command-deck`
   (public or private, doesn't matter — Actions works either way).
2. Open the new repo → **Add file → Upload files**.
3. On your Chromebook, open the Files app, select **everything inside this
   `techaurix-android` folder** (including the hidden `.github` folder — you
   may need to drag it in separately since some browsers hide dotfolders),
   and drag it into the GitHub upload box. GitHub preserves the folder
   structure automatically.
4. Commit directly to `main`.

   *(Alternative if drag-and-drop misses `.github`: open
   `.github/workflows/build-apk.yml` here, then on GitHub use
   **Add file → Create new file**, type the path
   `.github/workflows/build-apk.yml` in the name field — GitHub auto-creates
   the folders — and paste the contents in.)*

## 2. Let Actions build the APK

1. On GitHub, open the **Actions** tab. A "Build APK" run should already be
   in progress (triggered by your push to `main`).
2. Wait for the green checkmark (first run takes a few minutes — later runs
   are faster).
3. Click into the finished run → scroll to **Artifacts** →
   download `techaurix-debug-apk.zip` → unzip it → you have `app-debug.apk`.

## 3. (Optional) Get a proper Release instead of a build artifact

Artifacts expire after 90 days. For a permanent download link:

1. On GitHub, go to **Releases → Draft a new release**.
2. Create a new tag like `v1.0` and publish it.
3. The same workflow re-runs, and because the tag matches `v*`, it attaches
   `app-debug.apk` directly to that Release page — a stable link you can
   share or bookmark on your phone.

## 4. Install on your phone

1. Download the APK on your Android phone (from the Release page or the
   Actions artifact).
2. Open it — Android will prompt to allow installs from that source
   (Chrome/Files) the first time. Approve it, then install.
3. Launch **TechAurix** — it loads the command deck from the bundled HTML
   file, so it works even without internet, as long as your phone is on the
   same Wi-Fi as the arm/drive/cam/sensor ESP32 modules.

## Notes

- This is a **debug-signed** APK — perfectly fine for installing on your own
  device, but Android will always show it as coming from an unverified
  developer. That's expected and fine for personal use.
- `network_security_config.xml` explicitly allows cleartext HTTP, since your
  ESP32 modules serve plain HTTP/WS on the local network, not HTTPS.
- If `armcam.local` style mDNS names don't resolve from inside the app later,
  that's the known Android WebView limitation you'd already been looking
  into — the fix is adding an `NsdManager`-based resolver in `MainActivity.kt`
  and exposing it to the page via a `WebView.addJavascriptInterface()`
  bridge. Happy to build that out next if mDNS resolution turns out to be
  flaky from inside the APK.
- To change the app name or package id, edit `applicationId`/`namespace` in
  `android/app/build.gradle` and `android/app/src/main/AndroidManifest.xml`.
