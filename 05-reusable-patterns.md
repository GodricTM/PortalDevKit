# 05 — Reusable patterns

Drop-in building blocks already written and shipped in `immortal/` and `PortalOverlays/`. Each entry:
what it's for, where the real code is, and the Portal-specific gotcha that makes it non-obvious.

---

## A. OTA auto-updater (the flagship)

**What:** the app checks a hosted `version.json` on launch, notifies if a newer build exists, and
installs it **in-app** — silently if a shell daemon is running, else via a one-tap system installer.
No app store, no cable.

**Code:** [`PortalOverlays/.../UpdateChecker.kt`](https://github.com/GodricTM/PortalOverlays/blob/main/app/src/main/java/com/portal/overlays/UpdateChecker.kt),
[`UpdateInstallReceiver.kt`](https://github.com/GodricTM/PortalOverlays/blob/main/app/src/main/java/com/portal/overlays/UpdateInstallReceiver.kt),
[`InstallDaemon.kt`](https://github.com/GodricTM/PortalOverlays/blob/main/app/src/main/java/com/portal/overlays/InstallDaemon.kt).
Immortal has the same approach for its launcher self-update.

**Flow:**
1. Host `version.json` on `main` (see [04](04-store-and-discovery.md)) with `versionCode / versionName / apkUrl / notes`.
2. On launch: `UpdateChecker.autoCheck(context)` — fetches the manifest **cache-busted**
   (`?t=<millis>`, so GitHub's raw CDN doesn't serve a stale copy), compares `versionCode` (long),
   posts a notification if newer. The notification opens the app (not a browser) to update in-app.
3. Manual "Check for updates" → `checkForUpdate(context) { result -> … }` returns a sealed
   `UpdateResult` (`Available` / `UpToDate` / `Failed`) you render however you like.
4. `installUpdate(context, info) { status -> }` downloads the APK to `cacheDir`, then:
   - if `InstallDaemon.isAvailable()` → silent `pm install` via the daemon;
   - else → `PackageInstaller` MODE_FULL_INSTALL session → the system "Install" confirmation, with the
     result delivered to `UpdateInstallReceiver`.

**Gotchas:**
- Needs `REQUEST_INSTALL_PACKAGES`. The same signing key must be used across releases or the update is
  rejected as a different app.
- `parseManifest` / `shouldUpdate` / `cacheBust` are factored as pure functions — easy to unit-test.

## B. Silent install daemon

**What:** lets the app `pm install` with **no on-device dialog** — because the shell user (uid 2000)
can install silently and a sideloaded app can't. A tiny shell script run once over ADB watches a queue
folder and installs anything dropped in.

**Code:** [`PortalOverlays/installd.sh`](https://github.com/GodricTM/PortalOverlays/blob/main/installd.sh), started by
`enable_portal_permissions`. App side: `InstallDaemon` drops the APK in
`/sdcard/Android/data/<pkg>/files/installq` and reads a `.heartbeat` to know the daemon is live.

**Gotchas:**
- It does **not** survive a reboot (it's a foreground shell process) — re-run the helper to restart.
  The app must fall back to `PackageInstaller` when the heartbeat is stale.
- Handing `pm` a `/sdcard` file descriptor fails with *"Failed transaction"* — stage the APK in
  `/data/local/tmp` first, then `pm install -r` that. (This is baked into `installd.sh`.)

## C. Draw-over-other-apps overlay service

**What:** a foreground service that owns a system-overlay surface and draws widgets/banners/HUD on top
of any app. Powers Portal Overlays' entire feature set.

**Code:** [`PortalOverlays/.../OverlayService.kt`](https://github.com/GodricTM/PortalOverlays/blob/main/app/src/main/java/com/portal/overlays/OverlayService.kt).

**Pattern highlights worth copying:**
- **`safeAddView`** wraps `WindowManager.addView` in try/catch. If "draw over other apps" is missing or
  revoked, `addView` throws `BadTokenException`/`SecurityException` — swallow it and report failure so
  the service **stays alive** instead of crashing. Always gate on `Settings.canDrawOverlays(this)`
  first and surface a "needs permission" state in the foreground notification.
- **Window params:** `TYPE_APPLICATION_OVERLAY` + `FLAG_NOT_TOUCH_MODAL | FLAG_NOT_FOCUSABLE`,
  `PixelFormat.TRANSLUCENT`, and `softInputMode = SOFT_INPUT_ADJUST_NOTHING`. Drop
  `FLAG_NOT_FOCUSABLE` only for overlays that need input (full-screen modals).
- **Draggable widgets:** a touch listener that moves the window via `updateViewLayout`, treats
  movement under an 8 dp slop as a tap, and persists positions. Re-clamp on rotation in
  `onConfigurationChanged` so nothing strands off-screen. (See `makeDraggable` / `clampToScreen`.)
- **Live refresh:** read all appearance from `Prefs` on every `ACTION_REFRESH` so settings apply live.
- **No ghost trails:** do **not** set `FLAG_LAYOUT_NO_LIMITS`, and set `softInputMode =
  SOFT_INPUT_ADJUST_NOTHING` on overlay windows — otherwise moving/dragging or a keyboard pan smears
  ghost copies that eat touches on Portal's compositor. See [06 § overlay trails](06-gotchas.md).

## D. Accessibility nav cluster (Back / Home / Recents / Lock)

**What:** the only sanctioned way for a normal app to fire system navigation. Floating buttons call
`performGlobalAction(...)`.

**Code:** [`PortalOverlays/.../NavAccessibilityService.kt`](https://github.com/GodricTM/PortalOverlays/blob/main/app/src/main/java/com/portal/overlays/NavAccessibilityService.kt)
(+ the manifest service tag and `@xml/nav_accessibility` config in [02](02-manifest-tags.md)).
Immortal has a back-only variant (`ImmortalBackGestureService`).

**Actions:** `GLOBAL_ACTION_BACK / HOME / RECENTS / LOCK_SCREEN (API 28+)`, plus a custom
Control-Center pull via `dispatchGesture` (swipe down from the top edge). Note: `TOGGLE_SPLIT_SCREEN`
returns `false` on Portal (no multi-window) — don't ship it.

> **Reinstalling disables the service.** Every `pm install -r` makes Android turn the accessibility
> service back off — re-grant `enabled_accessibility_services` after each update ([06](06-gotchas.md)).

**Gotcha — Recents on Portal Mini:** smaller Portals have **no Overview/Recents screen**, so
`GLOBAL_ACTION_RECENTS` returns `false` and does nothing. Distinguish "accessibility service not
enabled" (tell the user how to enable it) from "enabled but the action returned false" (the device has
no such feature). Portal Overlays falls back to its **own installed-apps switcher grid** in that case
(`showAppSwitcher()` — `queryIntentActivities` for `MAIN/LAUNCHER`, tap to `startActivity`).

## E. Keyless push — ntfy.sh

**What:** real-time push without FCM. Subscribe to an ntfy topic over SSE; each published message pops
a banner. Self-hostable.

**Code:** [`PortalOverlays/.../NtfyClient.kt`](https://github.com/GodricTM/PortalOverlays/blob/main/app/src/main/java/com/portal/overlays/NtfyClient.kt).
Publish from anywhere: `curl -d "Kitchen timer done" ntfy.sh/<your-topic>`.

## F. Keyless weather — Open-Meteo

**What:** current conditions, 15-minute precipitation (rain-in-next-hour), and sunrise/sunset — no API
key, no GMS. Geocode a city once, then poll.

**Code:** [`PortalOverlays/.../WeatherClient.kt`](https://github.com/GodricTM/PortalOverlays/blob/main/app/src/main/java/com/portal/overlays/WeatherClient.kt).
Endpoints: `geocoding-api.open-meteo.com/v1/search`, `api.open-meteo.com/v1/forecast` with
`current=…&minutely_15=precipitation&daily=sunrise,sunset&timezone=auto`.

**Gotchas:**
- With `timezone=auto`, timestamps are **location-local**; convert to UTC epoch using the returned
  `utc_offset_seconds` before comparing to `System.currentTimeMillis()`.
- Return raw epochs to the UI and recompute "in X min" each tick — gives a smooth countdown without
  re-fetching every second. (See `WeatherClient.Extras`.)

## G. Real-feed ticker overlay

**What:** a thin scrolling strip backed by real user-supplied data. No demo headlines, no bundled
placeholder feed: if the user has not configured a URL, nothing scrolls.

**Code:** [`PortalOverlays/.../TickerClient.kt`](https://github.com/GodricTM/PortalOverlays/blob/main/app/src/main/java/com/portal/overlays/TickerClient.kt).

**Pattern:** accept RSS, Atom, or JSON URLs; auto-detect by response body; extract short display strings
from common JSON fields (`title`, `headline`, `name`, `text`, `message`, `summary`) or XML `<title>`
tags. Poll every few minutes on a daemon thread and cap the number of displayed items.

**Gotcha:** keep networking off the UI thread and make the empty state explicit. The ticker should be a
real-data surface, not a fake-news or sample-data surface.

## H. Alert sounds via system ringtone picker

**What:** per-kind alert tones without bundling licensed audio files.

**Code:** `SoundRow(...)` in [`PortalOverlays/.../MainActivity.kt`](https://github.com/GodricTM/PortalOverlays/blob/main/app/src/main/java/com/portal/overlays/MainActivity.kt)
and `playAlertSound(...)` in [`OverlayService.kt`](https://github.com/GodricTM/PortalOverlays/blob/main/app/src/main/java/com/portal/overlays/OverlayService.kt).

**Pattern:** launch `RingtoneManager.ACTION_RINGTONE_PICKER`, store the returned URI per alert kind,
and fall back to `Settings.System.DEFAULT_NOTIFICATION_URI` when no custom tone is set.

## I. On-device permission onboarding + PC helper

**What:** Portal's Settings UI can't toggle some permissions, and owners aren't all technical. Two-part
solution: a first-run walkthrough that deep-links to the system pages it *can* open, and a
`.bat`/`.ps1` helper that grants the rest over adb.

**Code:** the `OnboardingOverlay` / `PermStep` composables in
[`PortalOverlays/.../MainActivity.kt`](https://github.com/GodricTM/PortalOverlays/blob/main/app/src/main/java/com/portal/overlays/MainActivity.kt),
and [`enable_portal_permissions.ps1`](https://github.com/GodricTM/PortalOverlays/blob/main/enable_portal_permissions.ps1) /
[`.bat`](https://github.com/GodricTM/PortalOverlays/blob/main/enable_portal_permissions.bat). Re-read permission state on
`Lifecycle.Event.ON_RESUME` (the user grants on a system page, then presses Back).

**Gotcha:** some Portal builds gate the per-app overlay settings page — fall back from
`ACTION_MANAGE_OVERLAY_PERMISSION` with a `package:` URI to the global list.

## J. Screenshot via MediaProjection

**What:** capture the screen from a sideloaded app (no system screenshot API). One-time consent via a
transparent activity, then mirror the display into an `ImageReader` and save to `Pictures/Screenshots`.

**Code:** `startScreenshot()` / `capture()` in `OverlayService.kt` +
[`ScreenshotActivity.kt`](https://github.com/GodricTM/PortalOverlays/blob/main/app/src/main/java/com/portal/overlays/ScreenshotActivity.kt).
FGS type must include `mediaProjection`.

## K. Soft-keyboard on Portal (bug + fix)

**Symptom:** tapping a text field focuses it but no keyboard appears.

**Causes & fix:**
- Don't put `stateUnchanged` in `windowSoftInputMode` — it tells the system to **leave the IME
  hidden** on focus. Use `android:windowSoftInputMode="adjustResize"`.
- On Portal's older Android the IME also won't reliably auto-raise inside a scrolled Compose form.
  Request it explicitly on focus — via Compose's `LocalSoftwareKeyboardController.show()` **and** a
  posted platform `InputMethodManager.showSoftInput(view, SHOW_IMPLICIT)` (the posted platform call is
  what actually sticks). See the `Field` composable in `MainActivity.kt`.
- Verify on-device with `adb shell dumpsys input_method | grep "mShowRequested\|mInputShown"` — the
  `screencap` screenshot won't show the IME surface even when it's up.

---

## Quick "what do I lift for a new app?" index

| I want… | Take from |
|---|---|
| Self-update without a store | UpdateChecker + version.json (A), optionally the silent daemon (B) |
| Draw on top of other apps | OverlayService `safeAddView` + draggable pattern (C) |
| System Back/Home/Recents buttons | NavAccessibilityService + manifest tag (D), with the Recents fallback |
| Push notifications | NtfyClient (E) |
| Weather / sun / rain | WeatherClient (F) |
| Real scrolling ticker | TickerClient + ticker overlay pattern (G) |
| Per-kind alert sounds | Ringtone picker + stored sound URIs (H) |
| Grant Portal permissions for users | onboarding + enable_portal_permissions helper (I) |
| Screen capture | MediaProjection pattern (J) |
| Text input that works | the keyboard fix (K) |
| Get discovered | Immortal catalog submission ([04](04-store-and-discovery.md)) |
