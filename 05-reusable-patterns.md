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

**Blind-navigation assistant pattern:** this same service shape can support an audio-first Portal UI:
listen to `AccessibilityEvent`s, inspect focused/clickable `AccessibilityNodeInfo`s, speak the current
screen/selection through Android `TextToSpeech`, and use `performAction(...)`,
`performGlobalAction(...)`, or `dispatchGesture(...)` for navigation. Keep expectations realistic:
cross-app node labels vary, Portal has no GMS/TalkBack stack to lean on, and reinstalling still disables
the service until it is re-granted. Best first target is an Immortal/Portal launcher assistant with a
small overlay, big tactile zones, spoken app names, Back/Home, "what is on screen?", and "open <app>".

**AOA accessory input:** the Portal also supports Android Open Accessory mode, including AOA v2 and
HID keyboard input over accessory mode. That is useful for external control/input experiments, but it
is not the same thing as unlocking ADB. Treat AOA as an input channel, not a shell/debug channel.

**Portal ROM / firmware dump takeaway:** the published `aloha` firmware dump shows the standard Portal
app stack and system images, but not a normal "developer unlock" package. The useful distinction is:
`AOA` is a real accessory/input path; `ADB` is present in firmware but appears gated by boot/factory
state (`ro.boot.force_enable_usb_adb`, `vendor.sys.boot_mode`, FFBM/QMMI, and related init logic).
For accessibility work, that means the promising path is an app or accessory that improves input and
navigation, not a claim that the dump itself exposes ADB.

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

## L. Better (neural) TTS — sideload a sherpa-onnx engine, out-of-process ★

**Problem:** Portal ships **one** TTS engine — `com.facebook.aloha.fbttsservice` (Meta's Aloha voice) —
and **no GMS**, so there's no Google TTS and no second engine to pick a nicer voice from. That one
engine is the quality ceiling for `android.speech.tts.TextToSpeech`.

**Don't** bundle a neural runtime (Piper/sherpa-onnx) *inside* your app: a model download that
truncates on Portal's flaky Wi-Fi makes onnxruntime abort **natively (`SIGABRT`)**, which takes your
*whole process* down — and the runtime + model bloats the APK (and your self-update download) by
100–400 MB. (Immortal learned this the hard way.)

**Do** sideload a **separate** [k2-fsa sherpa-onnx TTS *engine* APK](https://k2-fsa.github.io/sherpa/onnx/tts/apk-engine.html).
Two reasons it works where in-process didn't:
1. **Out-of-process** — it runs as its own app/process, registered as a system TTS engine
   (`android.intent.action.TTS_SERVICE`). A bad model or native crash kills *that* process; your app
   just falls back to FbTtsService. Crash-isolated.
2. **Model bundled in the APK** (one voice per APK) — **no runtime download → no truncation.** The
   engine extracts it to `/sdcard/Android/data/com.k2fsa.sherpa.onnx.tts.engine/files/<model>/` on
   first run.

**Critical:** these k2-fsa engine APKs are **minSdk 21**, so they install on **gen-1 Portal/Portal+
(API 28)**. The polished alternative, [SherpaTTS (`org.woheller69.ttsengine`)](https://f-droid.org/packages/org.woheller69.ttsengine/),
has a nice model/voice manager but is **minSdk 29 → gen-1 Portal+ can't run it** (only gen-2 Portals /
Quest). Native arm64 sherpa-onnx itself runs fine on API 28 — verified on a Portal+ (Piper RTF ≈ 0.38,
~2.7× real-time; Kokoro heavier but usable).

**Install + make it the default (provision over USB, never Portal Wi-Fi):**
```bash
adb install -r -d sherpa-onnx-<ver>-arm64-v8a-eng-tts-engine-vits-piper-en_US-lessac-medium.apk
adb shell settings put secure tts_default_synth com.k2fsa.sherpa.onnx.tts.engine   # needs WRITE_SECURE_SETTINGS
```
Voice families, worst→best: Piper `*-medium` (one voice, fast, ~85 MB) → Piper `*-high` →
**Kokoro** (`kokoro-multi-lang`, **~103 voices**, most natural, ~360 MB, heavier). Romanian etc. exist
as Piper APKs (`ro_RO-mihai-medium`).

**Current Portal+ custom engine setup (device-verified):**
- Immortal uses Android `TextToSpeech`; the neural runtime stays out-of-process in
  `com.k2fsa.sherpa.onnx.tts.engine`.
- Staged models live under
  `/sdcard/Android/data/com.k2fsa.sherpa.onnx.tts.engine/files/<model>/`.
- The current staged set is **Kokoro v1.0 English** (`kokoro-multi-lang-v1_0`) plus
  **Romanian Mihai** (`vits-piper-ro_RO-mihai-medium`). Kokoro itself does not provide Romanian.
- Kokoro v1.1 is not used on this Portal because the staged build included Chinese-focused voices that
  should not appear in the family picker.
- Meta's original Portal TTS remains available as `com.facebook.aloha.fbttsservice`. Switch back with
  `adb shell settings put secure tts_default_synth com.facebook.aloha.fbttsservice`; switch to Sherpa
  with `adb shell settings put secure tts_default_synth com.k2fsa.sherpa.onnx.tts.engine`.
- App voice pickers must list all non-network voices, not only `Locale.getDefault()`, so Romanian remains
  selectable on an English Portal.
- Android may keep a stale `TextToSpeech.voice` when `setVoice(...)` returns `ERROR`. Immortal and the
  custom Sherpa engine therefore also pass/read a private synthesis param,
  `sherpa_voice_name`, so the engine can resolve the intended Kokoro speaker/Romanian voice directly.

**Stock k2-fsa prebuilt APK gotchas:**
- **One active model per install.** Every k2-fsa engine APK shares the package
  `com.k2fsa.sherpa.onnx.tts.engine`, so installing a second one **replaces** the first as the *active*
  model (the previous model dir lingers in `files/` but isn't reachable — the prebuilt has no
  model-picker). To switch language/family you **reinstall** the other APK. For multiple selectable
  models in one app you'd need the API-29 SherpaTTS, or a custom engine that enumerates `files/`.
- **Voice selection within a multi-speaker model** (e.g. Kokoro) is a **Speaker ID** field in the
  engine's own activity; single-speaker Piper models have none. Apps can also enumerate speakers via
  `TextToSpeech.getVoices()` and `setVoice(...)`.
- Ship the chosen engine APK in your **provisioning kit** so a fresh Portal gets the good voice over
  USB — don't rely on the in-app HuggingFace download over the Portal's connection.

---

## M. DreamService screensaver that survives the idle screen ★

**Why:** a floating overlay is hidden the instant the screen saver starts — the system composites the
dream over `TYPE_APPLICATION_OVERLAY` ([06](06-gotchas.md)). To keep a now-playing card / clock /
dashboard on the idle screen, **be** the screen saver.

**Shape (the key move):** put the whole view tree + data wiring in a **`Context`-based scene class**, so
the *same* code renders in the real Dream **and** in a normal Activity for an in-app **Preview** (the
preview doesn't touch the registered screen saver, so the idle screen keeps working).

```kotlin
class ScreensaverScene(private val ctx: Context, private val onExit: () -> Unit) {
    fun createView(): View { /* build background + clock + now-playing against ctx; tap → onExit() */ }
    fun start() { /* startMedia(); tick(); optional reactor */ }
    fun stop()  { /* remove callbacks; release media + webview */ }
    fun keepBright() = Prefs(ctx).screensaverKeepBright
}
class YourDreamService : DreamService() {           // the real screen saver
    private var scene: ScreensaverScene? = null
    override fun onAttachedToWindow() { super.onAttachedToWindow()
        isInteractive = true; isFullscreen = true
        scene = ScreensaverScene(this) { finish() }.also { isScreenBright = it.keepBright(); setContentView(it.createView()) } }
    override fun onDreamingStarted() { super.onDreamingStarted(); scene?.start() }
    override fun onDreamingStopped() { scene?.stop(); super.onDreamingStopped() }
    override fun onDetachedFromWindow() { scene?.stop(); scene = null; super.onDetachedFromWindow() }
}
class ScreensaverPreviewActivity : Activity() {     // in-app Preview, same scene
    private var scene: ScreensaverScene? = null
    override fun onCreate(b: Bundle?) { super.onCreate(b)
        window.addFlags(WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON)
        scene = ScreensaverScene(this) { finish() }.also { setContentView(it.createView()) } }
    override fun onResume() { super.onResume(); scene?.start() }
    override fun onPause()  { scene?.stop(); super.onPause() }
}
```

**Gotchas baked in (all in [06](06-gotchas.md)):**
- A fullscreen WebView background swallows the dismiss tap → put a top transparent clickable catcher
  over everything that calls `onExit()`.
- Hide the battery line when `BatteryManager.EXTRA_PRESENT` is false (mains Portals show a fake "0%").
- A real audio visualizer can't follow the music — animate on media-session `STATE_PLAYING` instead.
- Manifest tag + activation (`settings put secure screensaver_components …`) in [02](02-manifest-tags.md);
  if Immortal is installed, revoke its `WRITE_SECURE_SETTINGS` so it stops reclaiming the slot.

Reference impl: `PortalOverlays/.../ScreensaverScene.kt`, `NowPlayingDreamService.kt`,
`ScreensaverPreviewActivity.kt`, and the `set_screensaver.bat` helper.

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
| Better/neural voice (no GMS) | sideloaded sherpa-onnx TTS engine, out-of-process (L) |
| Content on the idle screen / a screen saver | DreamService + shared Context scene + preview Activity (M) |
| Get discovered | Immortal catalog submission ([04](04-store-and-discovery.md)) |
