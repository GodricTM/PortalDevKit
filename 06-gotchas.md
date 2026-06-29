# 06 — Portal gotchas & fixes (field notes)

Hard-won, device-verified quirks. Each one cost real debugging time on Portal hardware; check here
before re-deriving them.

## Overlay windows leave ghost trails when they move ★

**Symptom:** a `TYPE_APPLICATION_OVERLAY` widget (nav cluster, status strip, any draggable) smears
**ghost copies** across the screen when it moves — i.e. when you **drag** it, when the layout changes,
or when a **soft keyboard opens** (the system pans the overlay up). The ghosts are real, touchable
windows' leftover pixels and they **steal touches**, so the app feels frozen and you have to exit.

**Root cause (two compounding factors on Portal's old compositor):**
1. `FLAG_LAYOUT_NO_LIMITS` lets the window extend off-screen and **defeats damage-region clearing** —
   the vacated pixels aren't repainted, so the old frame lingers.
2. Overlay windows default to `softInputMode = adjust=pan`. When a keyboard opens in *any* app, the
   system **pans every overlay upward**, dragging the smear across the whole screen.

**Fix** (in the shared window-params builder — `OverlayService.baseParams()`):
```kotlin
// DON'T set FLAG_LAYOUT_NO_LIMITS. Clamp positions on-screen yourself instead.
var flags = FLAG_NOT_TOUCH_MODAL or FLAG_NOT_FOCUSABLE   // (drop NOT_FOCUSABLE for modal overlays)
WindowManager.LayoutParams(WRAP_CONTENT, height, TYPE_APPLICATION_OVERLAY, flags, TRANSLUCENT).apply {
    softInputMode = WindowManager.LayoutParams.SOFT_INPUT_ADJUST_NOTHING   // keyboard won't pan overlays
}
```
**Diagnose** with `dumpsys window windows | grep -A2 com.yourapp` — if there's only **one** window per
overlay but you *see* many on screen, it's a trail, not duplicate windows. The overlay's `mAttrs` will
show `sim={adjust=pan}` (the smoking gun) before the fix.

## Hardware mic-mute makes capture open but stay silent — dead-mic watchdogs thrash ★

**Symptom:** an `AudioRecord` watchdog (e.g. `VoiceBridge`) reports `rms≈1` (pure silence), logs
`stream silent …ms (dead mic); rebuilding`, then `closeMic` → `openMic` **in a loop every ~4s,
forever**. The open is *not* failing — `start_input_stream: exit` succeeds and an `AudioFlinger`
thread comes up each cycle; the stream just carries silence.

**Root cause:** Portal's **mic mute is a hardware switch**. When it's off, the signal is cut *before*
the audio path, so the capture stream opens normally but delivers ~0 RMS. A watchdog that equates
"sustained silence" with "broken mic" **false-positives on a completely normal Portal state** (user
muted the mic) and rebuilds endlessly — churning AudioFlinger threads and ACDB lookups every cycle.

**Fix** — don't treat silence as a dead mic while muted, and back off regardless:
```kotlin
// Gate the dead-mic check on mute state before rebuilding.
if (audioManager.isMicrophoneMute()) return            // muted, not broken — do nothing
// also listen for AudioManager.ACTION_MICROPHONE_MUTE_CHANGED to re-arm
// and cap rebuilds (exponential backoff) so genuine silence can't churn the HAL at 4s intervals
```
If `isMicrophoneMute()` doesn't reflect the **hardware** switch on your model, treat sustained
silence as *muted* (pause + surface a "mic off" hint) rather than *dead* — never loop a HAL rebuild on it.

**Not the cause (log noise that rides along):** `avc: denied { write } … hal_audio_default …
fifo_file`, `ACDB-LOADER … Returned = -19`, and `Request requires …RECORD_AUDIO_PRIVILEGED` all show
up in the same trace but are unrelated to the muted silence — don't chase them first.

**Evidence:** observed in device logcat (rms=1 rebuild loop); confirmed by toggling the hardware mic switch.

## Soft keyboard won't show / breaks touch

**Symptom:** tapping a text field focuses it but no keyboard appears; or it appears but the app stops
responding to touch.

**Causes & fixes:**
- **`windowSoftInputMode="stateUnchanged"` actively suppresses the IME** on focus. Use plain
  `adjustResize`.
- On Portal's old Android the IME also doesn't reliably auto-raise inside a scrolled Compose form.
  Request it explicitly on focus — Compose's `LocalSoftwareKeyboardController.show()` **and** a posted
  platform `InputMethodManager.showSoftInput(view, SHOW_IMPLICIT)` (the posted platform call is what
  actually sticks).
- If touch then "freezes," it's almost always the **overlay ghost-trail bug above** (the smeared HUD
  windows over the form eat taps), not the IME itself. Fix that and typing works.
- **Verify on-device with `dumpsys input_method | grep "mShowRequested\|mInputShown"`** — `screencap`
  (what `metavr capture screenshot` falls back to on Portal) **does not composite the IME surface**, so
  the keyboard won't appear in a screenshot even when it's genuinely up.

## Reinstalling the app disables its accessibility service ★

Every `metavr app install -r` (app update) makes Android **disable the app's accessibility service**
as a security measure — so the nav cluster (Back/Home/Recents/Lock) silently stops working and the
in-app "ACCESSIBILITY" indicator goes dark. **Re-enable after every reinstall:**
```bash
metavr adb shell settings put secure enabled_accessibility_services com.yourapp/com.yourapp.NavAccessibilityService
metavr adb shell settings put secure accessibility_enabled 1
```

## Recents / Overview — Portal Mini has none; Portal+ does

`performGlobalAction(GLOBAL_ACTION_RECENTS)`:
- **Portal+** — works; opens Portal's **native (Facebook) overview** UI.
- **Portal Mini (incl. 2nd gen) / smaller models** — returns **`false`** and does nothing: their
  appliance SystemUI **ships no Overview/Recents activity** (smart-display launcher, no multitasking
  switcher). This is a platform limitation, not an app bug.

Handle it: distinguish "accessibility service off" from "action returned false while enabled", and on
`false` fall back to your own installed-apps switcher (`queryIntentActivities(MAIN/LAUNCHER)` → tap to
`startActivity`). See `OverlayService.showAppSwitcher()`.

## Split-screen is not available

`GLOBAL_ACTION_TOGGLE_SPLIT_SCREEN` returns `false` on Portal — the appliance SystemUI has no
split-screen/multi-window mode. Don't ship it as a feature (we tried, then removed it).

## `screencap` is the only screenshot path

`metavr capture screenshot` falls back to `screencap` (no `com.oculus.metacam` on Portal). It works for
app UI but **omits the IME surface** and some secure/DRM surfaces. For verifying keyboard/overlay state,
trust `dumpsys` over the picture.

## Windows USB driver — Code 10 after install

After installing the patched WinUSB driver, the ADB interface often shows **Problem Code 10 /
`STATUS_NO_SUCH_DEVICE`** ("cannot start") — a stale enumeration, **not** a signature block (that's
Code 52). **Unplug, toggle ADB Enabled off/on on the Portal, replug** to re-enumerate. The device may
also alternate between enumerating as `Portal` (single ADB interface, `PID_1800`) and `Portal+`
(composite, `PID_1801`) — both are fine once a driver is bound. Full driver walkthrough in
[03 § USB-driver fix](03-build-and-sideload.md).

If you are trying to control the Portal without ADB, remember there are two different USB paths:
`AOA`/accessory mode and `ADB`. AOA can accept HID keyboard input and accessory strings, but it does
not grant shell access. ADB is present in firmware and appears to be gated by boot/factory logic
(`ro.boot.force_enable_usb_adb`, `vendor.sys.boot_mode`, FFBM/QMMI, and related init scripts), not by
ordinary app settings.

If you are working from the published `aloha` ROM dump, do not assume it contains a bypass or hidden
debug app. The dump is still useful as evidence of the real boot and accessory plumbing, but not as a
simple unlock package.

## Test harness mode

`adb shell cmd testharness enable` appears to be a useful device-local provisioning mode on Portal.
Community reports say it resets the device while preserving ADB, skips part of first-run setup, and can
clear or bypass the normal 4-digit lock screen during that mode. It may also leave the device ready for
a replacement launcher over ADB, which is why it is interesting for Portal recovery and accessibility
work.

Treat it as a provisioning/reset path, not as a full factory unlock. Verify the exact side effects on
your own unit before relying on it, especially if you are trying to preserve installed apps or account
state.

## A full-screen WebView swallows tap-to-exit (`setOnTouchListener { false }` can't fix it) ★

**Symptom:** a full-screen **WebView** layered over a tap-to-dismiss surface — an HTML clock face or
dashboard on top of a `DreamService` screensaver or a holding Activity — makes **tap-to-exit and
swipe dead**. The host's `root.setOnTouchListener` never even logs the touch, so the screensaver/HUD
can't be dismissed and reads as frozen/stuck. A *non-WebView* face over the same host dismisses fine.

**Root cause:** a WebView is inherently clickable/scrollable, so **`WebView.onTouchEvent()` returns
`true` and consumes the event**. The common "let it pass through" attempt —
`setOnTouchListener { _, _ -> false }` — does **not** work: a listener returning `false` only means
"I didn't handle it," after which the View's own `onTouchEvent` still runs and eats the touch. So the
WebView consumes it *before* it can bubble up to the parent/host listener.

**Fix** — subclass and return `false` from `onTouchEvent` (and `performClick`) so touches fall
through to the host:
```kotlin
val web = object : WebView(context) {
    override fun onTouchEvent(event: MotionEvent): Boolean = false
    override fun performClick(): Boolean = false
}
```
Touch dispatch now falls through WebView → parent → the host's `OnTouchListener`, restoring
tap-to-exit / swipe-to-change. Only do this when the page is **non-interactive** (a self-ticking
clock, a static dashboard); if the page needs scrolling or links, gate it by region/state instead.

**Diagnose:** with the WebView up, `adb shell input tap X Y` produces **no** touch log in the host and
`dumpsys window | grep mCurrentFocus` stays on the screensaver/overlay window. Swap the WebView face
for a plain one and the same tap dismisses instantly — that isolates it to the WebView, not the host.

**Evidence:** Portal+ `DreamService` photo frame with an HTML flip-clock face — trap and fix both
verified on device (tap → `onExit → finish()` → `USER_EXIT` only after the override).

## Real-time audio capture (output-mix or mic) is not viable for a sideloaded app ★

**Goal that fails:** a music **visualizer that reacts to the actual audio** (FFT/levels of what's
playing). Both Android capture paths are blocked or degraded on Portal — verified on Portal+ (API 28).

1. **Output mix via `Visualizer(0)` — BLOCKED.** Constructing `android.media.audiofx.Visualizer` on
   session `0` (the global mix) fails immediately:
   ```
   E visualizers-JNI: Visualizer initCheck failed -3
   E Visualizer-JAVA: Error code -3 when initializing Visualizer.
   ```
   `-3` = `ERROR_INVALID_OPERATION`. Capturing the system mix needs the **privileged, signature-level
   `android.permission.CAPTURE_AUDIO_OUTPUT`**, ungrantable to a sideloaded app. `RECORD_AUDIO` is not
   enough. There's also no public way to get another app's audio **session id** (e.g. Spotify's) to
   attach a per-session `Visualizer`, so you can't target the player's stream either.

2. **Microphone via `AudioRecord` — works, but laggy/choppy.** `AudioRecord(MIC, 44100, MONO)`
   initializes, starts, and **reads** (the mic hears the speakers), so a Goertzel/FFT band bank does
   move. But Portal's **always-on assistant owns the single near-field mic and preempts your stream**:
   ```
   W AudioRecord: dead IAudioRecord, creating a new one from obtainBuffer()
   E         : Request requires com.facebook.alohasdk.permission.RECORD_AUDIO_PRIVILEGED
   V AudioPolicyManagerCustom: startInput(...) stopping silenced input 5238
   ```
   Your non-privileged input is repeatedly **silenced + killed + recreated**, delivering data in bursts
   with multi-hundred-ms gaps → visible lag/stutter. View-side easing smooths motion but can't fill the
   gaps. The uninterrupted/far-field path is behind `…RECORD_AUDIO_PRIVILEGED` (Meta-signed).

**What to ship instead:** treat the **media session's `STATE_PLAYING`** as the reactive signal and run a
**smooth synthetic** visualizer (animate while playing). Keep any mic reactor opt-in + off by default,
guarded so a refused/contended capture returns `false` and falls back. Full write-up:
`PortalOverlays/docs/portal-audio-capture.md`. (Distinct from the mic-mute loop above: that's the
hardware mute switch; this is privileged-listener contention + an output-mix permission wall.)

## A running screensaver (Dream) composites OVER your `TYPE_APPLICATION_OVERLAY` — re-host as a Dream ★

**Symptom:** floating overlays (now-playing, status strip, nav) **vanish the moment the screen saver /
Daydream starts**, then return when it ends. The views aren't being removed — it's pure z-order.

**Root cause:** `TYPE_APPLICATION_OVERLAY` is the highest window type a sideloaded app may use, and the
system draws a **`DreamService`** above it. By design, nothing an overlay does can paint over a screen
saver. (This is also why a launcher's *own* now-playing survives its dream — it's drawn **inside** the
dream's view tree, not as a separate window.)

**Fix:** to keep content on the idle screen, **ship your own `DreamService`** and have the user select it
as the device screen saver. Build the view tree against a plain `Context` so the same scene can be
hosted by both the `DreamService` and a normal Activity (for an in-app **Preview** — a preview Activity
does not disturb the registered screen saver). Activate without the (often-hidden) system picker via:
```bash
adb shell settings put secure screensaver_components com.yourapp/.YourDreamService
adb shell settings put secure screensaver_enabled 1
adb shell am start -n com.android.systemui/.Somnambulator   # trigger the active dream to test
```
See the reusable pattern in [05 § DreamService screensaver](05-reusable-patterns.md).

## The Immortal launcher reclaims the screensaver slot on boot / return-home ★

If the device runs the **Immortal launcher**, its `SettingsGuard.reaffirmScreensaver` **re-asserts its
own `screensaver_components`** on boot and on every return to its home screen (it forces
`screensaver_enabled=0` if *its* saver is off, or overwrites the component if on) — so any other screen
saver you register is **evicted**. That self-healing is a **no-op without `WRITE_SECURE_SETTINGS`**, so:
```bash
adb shell pm revoke com.immortal.launcher android.permission.WRITE_SECURE_SETTINGS   # then register yours
# undo: adb shell pm grant com.immortal.launcher android.permission.WRITE_SECURE_SETTINGS
```
Verify it holds by reading `settings get secure screensaver_components` **after** bouncing through the
launcher's home.

## Mains-powered Portals report no battery — hide "0%" ★

The Portal+ (and other plugged-in models) carry **no battery**: the `ACTION_BATTERY_CHANGED` sticky
intent returns `EXTRA_PRESENT=false` and `EXTRA_LEVEL=0`, so a naive readout shows a misleading
**"0%"**. Gate any battery UI on `getBooleanExtra(BatteryManager.EXTRA_PRESENT, false)` and hide it when
absent.

## You can't `am start` a non-exported activity from adb on Portal ★

`adb shell am start -n com.yourapp/.SomeActivity` for an `exported="false"` activity fails with
`Permission Denial: … not exported from uid …` — the Portal shell uid lacks `START_ANY_ACTIVITY`. To
**test** an internal activity from the CLI, temporarily set `android:exported="true"`, verify, then
revert (keep it `false` in the shipped build). Dreams are testable without this via the `Somnambulator`
trick above.

## Make user-facing `.bat` helpers `pause` at the end ★

A double-clicked `.bat` closes its console instantly, so users never see the success/failure output.
End wrapper scripts with `echo.` + `pause` so the window holds until a keypress. (From a terminal it
just waits for one key — harmless.)

## Quick triage table

| Symptom | Likely cause | Fix |
|---|---|---|
| HUD widgets smear / duplicate when dragged or typing | overlay trail bug | `SOFT_INPUT_ADJUST_NOTHING` + drop `FLAG_LAYOUT_NO_LIMITS` |
| App "frozen", must exit, after tapping a field | ghost overlays eating touches | same fix as above |
| Mic capture silent (rms≈0), watchdog rebuilds in a loop | hardware mic-mute switch is off | gate dead-mic check on `isMicrophoneMute()` + back off rebuilds |
| Keyboard never appears | `stateUnchanged`, or no explicit IME show | `adjustResize` + `showSoftInput` on focus |
| Nav buttons stopped working after an update | reinstall disabled accessibility | re-grant `enabled_accessibility_services` |
| Recents button does nothing (Mini) | no Overview screen on this model | app-switcher fallback |
| `metavr device list` empty though plugged in | USB driver / Code 10 | replug + toggle ADB; patched INF |
| Full-screen WebView (clock face / dashboard) can't be tapped away | `WebView.onTouchEvent` consumes the touch | subclass WebView → `onTouchEvent`/`performClick` return `false` |
| Audio visualizer won't react to the music | `Visualizer(0)` blocked (`-3`); mic preempted by assistant | use media-session `STATE_PLAYING` + synthetic animation; mic reactor opt-in only |
| Overlays vanish when the screen saver starts | a Dream composites over `TYPE_APPLICATION_OVERLAY` | ship your own `DreamService`; preview via a normal Activity |
| Your screen saver keeps getting reverted | Immortal's `reaffirmScreensaver` reclaims the slot | `pm revoke com.immortal.launcher …WRITE_SECURE_SETTINGS` |
| Battery shows "0%" on a Portal+ | mains-powered, no battery present | gate on `BatteryManager.EXTRA_PRESENT` |
| `am start` an internal screen fails (`not exported`) | shell lacks `START_ANY_ACTIVITY` | temporarily `exported=true` to test; Dreams via `Somnambulator` |

## Stock Portal behavior from the XDA thread

Forum posts reinforce a few behaviors that matter when debugging stock Portal:

- Setup was reported to require DNS for `www.facebook.com`, `graph.facebook.com`, and
  `graph-portal.facebook.com` at different stages.
- Browser and Bluetooth still worked on some units, while Alexa and the old "Hey Portal" path were
  dead or unreliable on later firmware.
- Spotify could appear in the app store even when it was missing from the music-services list, so
  app-store visibility and service integration were separate paths.
- The browser was reported as heavily throttled, with YouTube and Spotify becoming barely usable.
- The rear USB-C port was repeatedly described as a factory/debug path, which matches the AOA and
  ADB-gating behavior we already documented.
