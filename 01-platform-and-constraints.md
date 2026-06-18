# 01 — Platform & constraints

Meta Portal is a Snapdragon-based Android tablet/TV-stick family running a **modified AOSP without
Google Mobile Services**. Sales ended in 2022; ADB is open, so owners sideload their own apps.

## Device matrix

| Device | API (minSdk) | Form factor | Connection |
|---|---|---|---|
| Portal / Portal+ (1st gen, 2018) | 28 | Touch, landscape | USB-C (back) |
| Portal / Portal+ (2nd gen, 2019) | 29 | Touch, landscape | USB-C (back) |
| Portal Mini | 29 | Touch, portrait/landscape | USB-C (back) |
| Portal Go | 29 | Touch, battery | USB-C (under rubber flap) |
| Portal TV | 29 | TV stick, D-pad remote | USB-C |

- **`minSdk = 28`** covers everything. Use 29 if you don't need 1st-gen.
- **`targetSdk = 29`** for new apps (safest). Porting an existing app with a higher target usually
  works — don't downgrade unless you hit a concrete runtime issue.
- **arm64-v8a.** For multi-ABI dependencies, pin the arm64 build.

## What crashes on launch (no-GMS)

Hard dependencies on any of these will crash or no-op:

- Play Services, Firebase, **FCM** (push), Play Billing, Google Sign-In, Google Maps SDK, AdMob,
  ML Kit, Google Pay.
- `READ_CONTACTS` is **denied** at runtime. `AccountManager` returns nothing.

Keyless replacements we use (see [05](05-reusable-patterns.md)):

| Need | GMS thing to avoid | What we use instead |
|---|---|---|
| Push notifications | FCM | **ntfy.sh** (self-hostable, HTTP/SSE) |
| Weather | Google/paid APIs | **Open-Meteo** (no key) + IP geolocation |
| Maps/geocode | Maps SDK | Open-Meteo geocoding API |
| Crash/analytics | Firebase | none, or a self-hosted endpoint |

## Hardware capabilities

| Capability | Status on Portal |
|---|---|
| Camera (`CAMERA`) | Works (`Camera2`); framing/auto-pan is via the Smart Camera service, not raw `Camera2` |
| Microphone (`RECORD_AUDIO`) | Basic single-channel `handset-mic` works. The far-field beamformed array (Hey-Portal wake word) is gated behind a Meta-signed permission — **not** available to sideloaded apps |
| Speaker | No permission needed |
| Bluetooth / Wi-Fi / Internet | Standard permissions |
| Storage write | `WRITE_EXTERNAL_STORAGE` (Android-9 storage model) |
| Recents / Overview screen | **Not present on smaller models (e.g. Portal Mini).** `performGlobalAction(GLOBAL_ACTION_RECENTS)` returns `false` — there is no system overview UI to invoke. Provide a fallback (Portal Overlays falls back to its own app-switcher grid). |
| Contacts / Accounts | Not available |

## Design rules (tabletop, not phone)

Portal sits on a counter; users are 50–100 cm away.

- **Landscape-first.** (Portal Mini also does portrait.)
- **Hit targets ≥ 64 dp** (96 dp for primary actions). **Body text ≥ 16 sp** (18 sp on Portal+).
- **Reserve the top ~64 dp.** A persistent white system overlay strip floats above app content:
  back/home pills top-left, Wi-Fi/status top-right. It has **no safe-area inset** — edge-to-edge top
  bars tuck under it. If your top region is light-colored, add a dark scrim (the pills are white).
- **Dark UI** reads best on these panels and avoids fighting the white system pills.

## The "ground truth" tools

When in doubt about a Portal/Quest behavior, verify against the device instead of guessing:

- `metavr device list`, `metavr adb shell …`, `metavr adb logcat`, `metavr capture screenshot`.
- The bundled **`meta-vr:portal`** skill (loaded via the Skill tool) is the authoritative reference
  and pairs with this kit.
