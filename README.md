# TechAurix Command Deck

A browser-based control interface for a multi-module ESP32 robotics rig —
robotic arm, BLDC drill, drive base, live camera, and thermal/gas
sensing — packaged both as a static web app and as a sideloadable Android
APK.

Built for phone/Chromebook-only workflows: no PC, no Android Studio, no
desk-bound setup. Everything from editing to building the APK happens
through the browser or GitHub Codespaces.

---

## What it does

The command deck is a single-page HTML/CSS/JS app split into three panels:

| Panel | Control | Behavior |
|---|---|---|
| **Left — Arm** | Joystick (base + elbow), sliders (shoulder/wrist/grip), drill slider | Push the stick forward → elbow servo moves forward; pull back → it retracts. Left/right on the stick rotates the base, the same way the drive stick steers. Shoulder, wrist, and grip get individual sliders since one stick can't drive 5 axes at once. |
| **Center — Camera** | Live MJPEG feed, thermal overlay toggle, snapshot | Full-size video feed from the cam module, with an 8×8 thermal grid overlay and live MQ2 smoke / thermal-peak gauges underneath. |
| **Right — Drive** | Joystick (X = turn, Y = forward/back) | Standard tank-style two-axis stick for the drive base. |

A dedicated **BLDC EMERGENCY STOP** button zeros the drill and returns the
arm to its home position (90°/90°) instantly, independent of network
latency to any one module.

The layout is a responsive 3-column grid on wide screens (Chromebook,
laptop, desktop) that collapses to a single scrolling column — camera
pinned to the top — on narrow or portrait phones.

Node discovery (mDNS hostnames like `armcam.local`, manual IP entry, or a
subnet scan fallback) is handled through a **Link Setup** modal, opened via
the gear icon or any of the node status chips in the header.

---

## Architecture (hardware side — being confirmed)

The web app expects up to **4 independent network nodes**, each with its
own HTTP/WebSocket server:

| Node | Confirmed role | Talks to the app via |
|---|---|---|
| `cam` | ESP32-CAM — video stream | `GET /stream` (MJPEG) |
| `arm` | Arm servos + BLDC drill | WebSocket — `{axis, value}` per joint, `{estop:true}` |
| `drive` | Drive base motors | WebSocket — `{x, y}` |
| `sensors` | Thermal array + MQ2 smoke | `GET /sensors` (polled every 600ms) |

> ⚠️ **Not yet finalized.** The exact module split (e.g. whether drive and
> arm share one ESP32 or are separate, and which board owns the BLDC
> driver) is being confirmed against the actual firmware. This table — and
> the JS in `techaurix-app.html` that calls `sendWs('arm', …)` /
> `sendWs('drive', …)` — will be updated once the firmware is shared and
> checked against it.

Each node is configured independently in the Link Setup modal: mDNS
hostname, manual IP, or discovered via subnet scan. Connection state per
node shows live in the header chips (DRV / ARM / CAM / SNS) with an RSSI
bar.

---

## Repository structure

```
.
├── index.html                          # web app (GitHub Pages-hostable)
├── android/                            # Android WebView wrapper project
│   ├── build.gradle
│   ├── settings.gradle
│   ├── gradle.properties
│   └── app/
│       ├── build.gradle
│       └── src/main/
│           ├── AndroidManifest.xml
│           ├── assets/techaurix-app.html   # app bundled offline in the APK
│           ├── java/com/techaurix/commanddeck/MainActivity.kt
│           └── res/                        # icon, theme, network config
└── .github/workflows/build-apk.yml     # auto-builds the APK on every push
```

---

## Running it as a web app

No build step. Either:
- Open `index.html` directly (works offline, minus the live feed/sensor
  data which need the ESP32 nodes reachable on the same network), or
- Host it on **GitHub Pages**: repo → Settings → Pages → Deploy from
  branch → `main` / root.

---

## Building the Android APK

The APK is built entirely by GitHub Actions — no local Android SDK needed.

1. Push this repo to GitHub (see below for doing this from a Chromebook via
   Codespaces, with zero local git setup required beforehand).
2. GitHub Actions (`.github/workflows/build-apk.yml`) runs automatically on
   every push to `main`:
   - Sets up JDK 17 + Android SDK + Gradle
   - Runs `gradle assembleDebug` inside `android/`
   - Uploads `app-debug.apk` as a workflow artifact
3. Pushing a tag like `v1.0` additionally attaches the APK to a GitHub
   Release, giving you a permanent download link instead of a 90-day
   artifact.

### From a Chromebook, using GitHub Codespaces

```bash
# after uploading techaurix-android.zip into the Codespace file explorer:
unzip techaurix-android.zip

shopt -s dotglob
mv techaurix-android/* .
rmdir techaurix-android
shopt -u dotglob

rm techaurix-android.zip

git add .
git commit -m "Add TechAurix command deck APK build (WebView + GitHub Actions)"
git push origin main
```

Then check the **Actions** tab on the repo — the build runs automatically.

### Installing on your phone

Download `app-debug.apk` from the Actions artifact or the Release page,
open it on your phone, allow install from that source when prompted, and
install. The app loads the command deck from the bundled HTML — it doesn't
need internet, only Wi-Fi on the same LAN as the ESP32 modules.

---

## Notes & known limitations

- **Debug-signed only.** Fine for personal sideloading; Android will always
  flag it as an unverified developer since it isn't signed with a release
  key.
- **Cleartext HTTP is intentionally allowed** (`network_security_config.xml`)
  since the ESP32 modules serve plain HTTP/WS, not HTTPS.
- **mDNS resolution (`*.local` hostnames) inside the WebView is unverified.**
  Android's WebView networking stack doesn't reliably resolve mDNS the way
  a normal browser tab does. If `armcam.local`-style names fail from inside
  the APK, the fix is an `NsdManager`-based resolver in `MainActivity.kt`
  exposed to the page through `WebView.addJavascriptInterface()` — not yet
  implemented, planned once this is confirmed to be an actual problem.
- **Firmware/module mapping is provisional** — see the Architecture section
  above.

---

## Roadmap

- [ ] Confirm ESP32 module split and firmware endpoints against actual
      hardware
- [ ] Add `NsdManager` JS bridge if mDNS resolution proves unreliable
      in-app
- [ ] Release-sign the APK once the app is stable
- [ ] Optional: PWA manifest for "Add to Home Screen" as a lighter
      alternative to the APK on devices where sideloading is inconvenient
