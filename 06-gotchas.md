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
