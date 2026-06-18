# 03 — Build & sideload

## Toolchain

Two host pieces: a **JDK 17** (Gradle/AGP run on it) and the **Android SDK**.

- **JDK:** point `JAVA_HOME` at Android Studio's bundled JBR if installed —
  on this machine that's `C:\Program Files\Android\Android Studio\jbr`. Otherwise install Temurin 17.
- **SDK:** install via Google's `android` CLI (`platforms/android-28 android-29 platform-tools build-tools`).
- **metavr** (Meta VR CLI) — install via `npm i -g @meta-quest/metavr` or `npx -y metavr`. Use
  `metavr adb …` in place of raw `adb` everywhere.

## Gradle config

```kotlin
android {
    compileSdk = 35                 // current is fine
    defaultConfig {
        applicationId = "com.example.app"
        minSdk = 28                 // covers 1st-gen Portal/Portal+ and Portal TV
        targetSdk = 29              // Portal hardware tops out here
        versionCode = 1
        versionName = "1.0"
    }
    compileOptions { sourceCompatibility = JavaVersion.VERSION_17; targetCompatibility = JavaVersion.VERSION_17 }
    kotlinOptions { jvmTarget = "17" }
}
```

Release signing (Portal Overlays pattern — keystore + props stay **gitignored**):

```kotlin
val releaseProps = Properties().apply {
    rootProject.file("release-signing.properties").takeIf { it.exists() }?.let { FileInputStream(it).use(::load) }
}
signingConfigs { create("release") { /* storeFile/…/keyAlias from releaseProps */ } }
```

> **Keep the same signing key across releases.** A signed APK with the same key installs over the
> previous build (`-r`) and keeps user data. A debug-signed APK has a different key and forces an
> uninstall (wiping settings). For OTA self-update to work, every release must share the key.

## Build, install, launch, debug

```bash
# from Git Bash use the Android Studio JBR for JAVA_HOME:
JAVA_HOME="C:/Program Files/Android/Android Studio/jbr" ./gradlew assembleRelease

metavr device list                                   # confirm the Portal is seen
metavr app install -r app/build/outputs/apk/release/app-release.apk
metavr adb shell am start -n com.example.app/.MainActivity
metavr capture screenshot -o shot.png                # PNG of the screen (falls back to screencap)
metavr adb logcat *:E                                # errors only
metavr adb shell dumpsys package com.example.app | grep -E "versionCode|versionName"
```

> **Screenshot caveat:** `metavr capture screenshot` on Portal falls back to `screencap`
> (no `com.oculus.metacam`). `screencap` **does not composite the soft-keyboard (IME) surface**, so a
> raised keyboard won't appear in the capture even though it's on screen. Verify IME state with
> `adb shell dumpsys input_method | grep mInputShown` instead.

## Granting Portal permissions over adb

Some permissions can't be toggled in Portal's stripped Settings UI — grant them from the PC. These are
the ones Portal Overlays uses (see [`enable_portal_permissions.ps1`](https://github.com/GodricTM/PortalOverlays/blob/main/enable_portal_permissions.ps1)):

```bash
# Draw over other apps (overlays)
metavr adb shell appops set com.example.app SYSTEM_ALERT_WINDOW allow
# Accessibility service (global Back/Home/Recents)
metavr adb shell settings put secure enabled_accessibility_services com.example.app/com.example.app.NavAccessibilityService
metavr adb shell settings put secure accessibility_enabled 1
# Notification listener (now-playing / mirroring)
metavr adb shell cmd notification allow_listener com.example.app/com.example.app.NotifyListenerService
# Install permission (self-update)
metavr adb shell pm grant com.example.app android.permission.REQUEST_INSTALL_PACKAGES
```

Ship a `.bat` **and** `.ps1` helper so non-technical owners can run one file from a PC. Pin `*.sh`
helpers to **LF** via `.gitattributes` — CRLF breaks `/system/bin/sh` on the device.

## Windows USB-driver fix (the one that eats an afternoon)

Portal's ADB interface is **`VID_2EC6&PID_1800`** (single interface) / **`PID_1801`** (composite,
shown as *"Oculus Composite ADB Interface"*). Stock Windows has no driver for it, so it appears with
**no driver bound** (yellow ⚠) and `metavr device list` shows nothing.

We keep a patched WinUSB INF at [`drivers/android_winusb.inf`](drivers/android_winusb.inf)
that adds those VID/PIDs. It's **unsigned** (patching breaks Google's signature) and has **no
catalog**, so:

1. Install via **Device Manager → the "Portal" device → Update driver → Browse → Let me pick →
   Have Disk →** point at `android_winusb.inf` → choose **Android ADB Interface** → accept the
   "can't verify publisher" warning.
2. If a hard *signature* error blocks it: Settings → System → Recovery → **Advanced startup** →
   Troubleshoot → Advanced → **Startup Settings** → Restart → press **7** (*Disable driver signature
   enforcement*), then redo step 1.
3. **Bulletproof alternative:** [Zadig](https://zadig.akeo.ie) → *Options → List All Devices* →
   select the Portal interface → driver **WinUSB** → *Replace Driver*. Its WinUSB is Microsoft-signed,
   so no enforcement gymnastics.

**Gotchas observed:**
- After install the interface may show **Problem Code 10 / `STATUS_NO_SUCH_DEVICE`** ("cannot start").
  This is a stale-enumeration issue, **not** a signature block (that's Code 52). Fix: **unplug, toggle
  ADB Enabled off/on on the Portal, replug** so the interface re-enumerates and binds the new driver.
- The toggle can race the USB connect — if not detected, flip **ADB Enabled** off/on again.
- After a Windows reboot the binding can drop; that's why the patched INF goes into the **driver
  store** so it sticks.
